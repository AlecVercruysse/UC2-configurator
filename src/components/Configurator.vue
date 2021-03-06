<script>
 import axios from 'axios';
 import rateLimit from 'axios-rate-limit';
 import JSZip from 'jszip';
 import Promise from 'promise';
 import FileSaver from 'file-saver';
 import ModuleConfigurator from "./ModuleConfigurator.vue"

 export default {
     name: "ConfiguratorPanel",
     data: function() {
         return {
             repo: "AlecVercruysse/UC2-GIT",
             modules: [],
             applications: [],
             rateLimitedAxios: rateLimit(axios.create(), { maxRequests: 60, perMilliseconds: 60*60*1000}),
             modulesInUse: [],
             selectedAppName: "None",
             selectedFilePaths: []
         }
     },
     computed: {
         selectedApp: function() {
             const selectedApp = this.applications.find(x => x.name == this.selectedAppName)
             return selectedApp
         },
         containsDuplicateFiles: function() {
             return this.selectedFilePaths.some(x => x.count > 1)
         }
     },
     methods: {
         getItems(that) {
             /* get all applications and modules without loading configs */
             const url = "https://api.github.com/repos/" + this.repo + "/git/trees/master?recursive=1"
             this.rateLimitedAxios.get(url).then(function (response) {
                 const modules = response.data.tree.filter(function (item) {
                     /* look for all ASSEMBLY folders which contain config.json, without loading the config */
                     const path = item.path.split('/')
                     return (path[path.length - 1] == "config.json" && path[path.length - 2].includes("ASSEMBLY"))
                 })
                 const applications = response.data.tree.filter(function (item) {
                     /* look for all ASSEMBLY folders which contain config.json, without loading the config */
                     const path = item.path.split('/')
                     return (path[path.length - 1] == "config.json" && path[path.length - 2].includes("APP"))
                 })
                 var computeProperties = function (list) {
                     /* just computes the name, and adds a skeleton "config" that will get filled out by loading later */
                     list.forEach(function (item) {
                         const path = item.path.split('/')
                         item.name = path[path.length - 2]
                         item.config = { description:"",
                                         loaded: false
                         } 
                     })
                 }
                 computeProperties(modules)
                 computeProperties(applications)
                 applications.unshift({
                     name: "None",
                     config: {
                         type: "application",
                         description: "No Application. Add all modules yourself.",
                         modules: []
                     }
                 })
                 that.modules = modules
                 that.applications = applications
                 that.$emit('loading-done')
             })
         },
         generateID() {
             /*https://gist.github.com/gordonbrander/2230317 */
             return '_' + Math.random().toString(36).substr(2, 9);
         },
         constructModulesInUse() {
             this.modulesInUse = []
             for(let i = 0; i < this.selectedApp.config.modules.length; i++) {
                 let newModule = JSON.parse(JSON.stringify(this.modules.find(x => x.name == this.selectedApp.config.modules[i].name)))
                 newModule.key = this.generateID()
                 newModule.fixedOptions = this.selectedApp.config.modules[i].fixedOptions
                 newModule.applicationSpecific = true
                 /*                  console.log("created app specific (fixed) module in constructModulesInUse(), pushing to modulesInUse", newModule) */
                 this.modulesInUse.push(newModule)
             }
         },
         getAppConfig(item) {
             /* we prefer to grab directly from raw.githubusercontent.com as not to use up our rate limits with the api */
             const url = "https://raw.githubusercontent.com/" + this.repo + "/master/" + item.path
             axios.get(url).then(function (response) {
                 item.config = response.data
                 item.config.loaded = true
                 /* app specific stuff that belongs in the handler: */
                 this.constructModulesInUse()
             }.bind(this))
         },
         checkAppConfigLoaded() {
             if (this.selectedApp.config.loaded === false) {
                 this.getAppConfig(this.selectedApp)
             } else {
                 this.constructModulesInUse()
             }
         },
         addModule() {
             let newModule = JSON.parse(JSON.stringify(this.modules[0]))
             newModule.key = this.generateID()
             newModule.applicationSpecific = false
             newModule.fixedOptions = {} /* should always be empty for a module created with this function */
             this.modulesInUse.push(newModule)
         },
         updateSelectedModule(module) {
             let oldModuleIndex = this.modulesInUse.findIndex(x => x.key === module.key)
             /* vue wont detect Object.assign since it changes the array at an index: https://vuejs.org/v2/guide/reactivity.html#For-Arrays */
             this.modulesInUse.splice(oldModuleIndex, 1, module)
         },
         updateSTLFileList() {
             this.selectedFilePaths = []
             for(let modIdx = 0; modIdx < this.modulesInUse.length; modIdx++) {
                 const module = this.modulesInUse[modIdx]
                 if (module.config.loaded) {
                     /* first load in fixed files:  */
                     module.config.fixedFiles.forEach( function (fpath) {
                         this.addFileToSTLFileList(fpath)
                     }.bind(this))
                     /* now load in dynamic files */
                     module.config.dynamicFiles.forEach( function (file) {
                         if (this.shouldIncludeFile(file, module)) {
                             this.addFileToSTLFileList(file.path)
                         }
                     }.bind(this))
                 }
             }
         },
         shouldIncludeFile(file, module) {
             for (let [option, choices] of Object.entries(file.conditions)) {
                 if (choices.includes(module.config.options[option].selected) === false) {
                     return false
                 }
             }
             return true
         },
         addFileToSTLFileList(fpath) {
             const splitFpath = fpath.split('/')
             const idx = this.selectedFilePaths.findIndex(f => f.displayName == splitFpath[splitFpath.length - 1])
             if (idx != -1) {
                 this.selectedFilePaths[idx].count ++
             } else {
                 const path = fpath.split('/')
                 this.selectedFilePaths.push({
                     path: '/' + fpath,
                     count: 1,
                     displayName: path[path.length - 1]
                 })
             }
         },
         deleteModule(module) {
             this.modulesInUse = this.modulesInUse.filter(x => x.key !== module.key)
         },
         downloadZIP() {
             this.$emit('loading')
             const zip = new JSZip()
             let promises = [] 
             this.selectedFilePaths.forEach(file => {
                 const url = "https://raw.githubusercontent.com/" + this.repo + "/master/" + file.path
                 promises.push(axios.get(url, {
                     responseType: 'blob'
                 }
                 ).then(response => {
                     console.log(response)
                     const content = new Blob([response.data])
                     zip.folder("UC2-STL").file(file.displayName, content, {binary: true})
                 }))
             })
             Promise.all(promises).then(() => zip.generateAsync({type:"blob"}).then(content => {
                 FileSaver.saveAs(content, "UC2-STL.zip")
                 this.$emit('loading-done')
             }))
         }
     },
     created: function() {
         this.getItems(this)
     },
     watch: {
         selectedAppName: function() {
             this.checkAppConfigLoaded()
         },
         modulesInUse: {
             deep: true,
             handler() {
                 this.updateSTLFileList()
             }
         }
     },
     components: {
         ModuleConfigurator
     }
 }
 
</script>

<template>
    <div>
        <div class="container card-body"><h3>UC2-Configurator <br><small class="text-muted"> Configure and download all required STL files </small></h3></div>
        <div class="row card-body">
            <div class="col-md-8 p-4"> <!-- ALL SELECTION ITEMS -->
                <form>
                    <div class="form-group">
                        <label for="applicationSelect" class="font-weight-bold"> Select Application </label>
                        <select class="form-control" id="applicationSelect" v-model="selectedAppName">
                            <option v-for="application in applications" :key="application.name"> {{ application.name }}
                            </option>
                        </select>
                        <small class="form-text text-muted" v-if="selectedApp"> {{ selectedApp.config.description }} </small>
                    </div>
                    <hr/>
                    <p class="font-weight-bold"> Configure Modules: <br>
                        <small class="text-muted"> Options in gray are fixed and required by the application's configuration. </small>
                    </p>
                    <module-configurator v-for="module in modulesInUse" :key="module.key" v-bind:providedModule="module" v-bind:modules="modules" v-bind:repo="repo" v-on:update-selected-module="updateSelectedModule($event)" v-on:delete-selected-module="deleteModule($event)"></module-configurator>
                    <hr>
                    <button type="button" class="btn btn-outline-primary"  v-on:click="addModule">Add Module</button>
                </form>
            </div>
            <div class="col-md-4 card p-4"> <!-- A LIST OF INCLUDED FILES -->
                <p class="font-weight-bold">Files Needed:</p>
                <ul class="list-group">
                    <li class="list-group-item small" v-for="item in selectedFilePaths" v-bind:key="item.path"> {{ item.count }}x {{ item.displayName }} </li>
                </ul>
                <hr>
                
                <button type="button" class="btn btn-primary" v-on:click="downloadZIP">Download ZIP</button>
                <p class="small text-danger my-4" v-if="containsDuplicateFiles">Warning: Your configure requires multiple copies of a file. The ZIP comes with a single copy of each. Please remember to print all files! </p>
            </div>
        </div>
    </div>
</template>


<style>
 @import'~bootstrap/dist/css/bootstrap.css'
</style>
