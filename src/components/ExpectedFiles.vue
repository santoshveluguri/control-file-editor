<template>
    <v-container>
        <v-row>
            <v-btn elevation="2" @click="newRecord">Create New Record</v-btn>
            <v-spacer></v-spacer>
            <v-btn elevation="2" @click="resetFile()">Reset Expected Files</v-btn>
        </v-row>
        <v-card>
            <v-card-title>
                Expected Files
                <v-spacer></v-spacer>
                <v-text-field
                    v-model="search"
                    append-icon="mdi-magnify"
                    label="Search"
                    single-line
                    hide-details
                    class="searchInput"
                ></v-text-field>
                <v-icon class="deleteIcon" :disabled="disableDelete" @click="deleteSelected">mdi-delete</v-icon>
            </v-card-title>
            <v-data-table
                v-model="selected"
                show-select
                :headers = "headers"
                :items = "items"
                :search="search"
                v-bind:items-per-page= "itemsPerPage"
                v-bind:page= "page"
                class="elevation-1"
                id="table"
                :key="reRender"
                @pagination="paginate"
            >
                <template v-slot:body="{ items }">
                    <tbody>
                        <tr v-for="item in items" :key="item.FileRoot">
                            <td><v-checkbox :value="item.FileRoot" v-model="selected"></v-checkbox></td>
                            <td @click="clicked(item, 'FileNameRegex', false)">{{ item.FileNameRegex }}</td>
                            <td @click="clicked(item, 'FileRoot', false)">{{ item.FileRoot }}</td>
                            <td @click="clicked(item, 'FileSchema', false)">{{ displayList(item.FileSchema, 'FileSchema', item) }}</td>
                            <td @click="clicked(item, 'RemappingRequired', false)">{{ item.RemappingRequired }}</td>
                            <td @click="clicked(item, 'Key_Fields', false)">{{ displayList(item.Key_Fields, 'Key_Fields', item) }}</td>
                        </tr>
                    </tbody>
                </template>
                
            </v-data-table>
        </v-card>


        <v-dialog v-model="dialog" width="500" v-bind:persistent="persistentDialog">
            <v-card>
                <v-card-title>
                    Edit {{editAttribute}}
                </v-card-title>
                <v-card-text>
                    <v-form ref="form">
                        <v-textarea v-if="showBigInput"
                        v-model="editedItem[editAttribute]"
                        auto-grow
                        autofocus
                        rows="1"
                        ></v-textarea>
                        <v-text-field v-else v-model="editedItem[editAttribute]"
                        :rules="rules"
                        hide-details="auto"
                        ></v-text-field>
                    </v-form>
                </v-card-text>
                <v-card-actions>
                    <v-btn
                        color="primary"
                        text
                        @click="resetAttribute"
                    >
                        Reset
                    </v-btn>
                    <v-spacer></v-spacer>
                    <v-btn
                        color="primary"
                        text
                        @click="saveAttribute"
                    >
                        Save
                    </v-btn>
                    </v-card-actions>
            </v-card>
        </v-dialog>
    </v-container>
</template>

<script>
export default {
    name: 'ExpectedFiles',
    props: ['portcos', 'portcoToEdit'],
    data: () => ({
        myFile: [],
        items: [],
        originalItems: [],
        myPortco: '',
        fileName: '',
        reRender: 0,
        search: '',
        rules: [],
        selected: [],
        dialog: false,
        itemsPerPage: 10,
        page: 1,
        pageCount: 1,
        itemToEdit: '',
        editedItem: '',
        editAttribute: '',
        showBigInput: false,
        persistentDialog: false,
        headers: [
            { text:'File Name Regex', value: 'FileNameRegex'},
            { text:'File Root', value: 'FileRoot'},
            { text:'File Schema', value: 'FileSchema', width: '35%'},
            { text:'Remapping Required', value: 'RemappingRequired'},
            { text:'Key Fields', value: 'Key_Fields', width: '25%'}
        ],
        listHeaders: ['FileSchema','Key_Fields'],
        
    }),
    computed: {
        disableDelete() {
            if (this.selected.length > 0) {
                return false
            }
            else {
                return true
            }
        }
    },
    methods: {
        forceRenderer() {
            this.reRender += 1
        },
        displayList(listToDisplay, attribute, item) {
            if(item != null) {
                if (attribute === 'FileSchema' && listToDisplay.length > 70) {
                    listToDisplay = listToDisplay.slice(0,listToDisplay.indexOf(" ", 70)).concat(" ...")
                }
                else if (attribute === 'Key_Fields' && listToDisplay.length > 50) {
                    listToDisplay = listToDisplay.slice(0,listToDisplay.indexOf(" ", 50)).concat(" ...")
                }
            }
            return listToDisplay
        },
        clicked(item, attribute, persistent) {
            this.persistentDialog = persistent
            this.itemToEdit = item
            this.editedItem = Object.assign({}, this.itemToEdit)
            this.editAttribute = attribute
            if (this.listHeaders.includes(this.editAttribute)) {
                this.showBigInput = true
            }
            else {
                this.showBigInput = false
            }
            this.rulesFunc()
            this.dialog = true
        },
        saveAttribute() {
            if (this.$refs.form.validate()) {
                this.dialog = false
                for (var i of this.items) {
                    if (i == this.itemToEdit) {
                        this.items[this.items.indexOf(i)] = this.editedItem
                    }
                }
                this.forceRenderer()
            }
        },
        resetAttribute() {
            this.editedItem[this.editAttribute] = this.itemToEdit[this.editAttribute]
        },
        newRecord() {
            var newItem = {}
            for (var header of this.headers) {
                if(header.value !== "Key_Fields") {
                    newItem[header.value] = ''
                }
            }
            newItem['Delimiter'] = "\t"
            newItem['HeaderRowsToRemove'] = "0"
            newItem['TrailingRowsToRemove'] = "0"
            newItem['Key_Fields'] = ""
            this.items.push(newItem)
            this.forceRenderer()
            if (Math.ceil(this.items.length / this.itemsPerPage) > this.pageCount) {
                this.pageCount += 1
            }
            this.page = this.pageCount
            this.clicked(newItem, 'FileRoot', true)
        },
        paginate(val) {
            this.page = val.page
            this.itemsPerPage = val.itemsPerPage
            this.pageCount = val.pageCount
        },
        resetFile() {
            this.items = this.originalItems
            this.forceRenderer()
            this.page = 1
        },
        deleteSelected() {
            var newItems = []
            for (var item of this.items) {
                if (!this.selected.includes(item.FileRoot)) {
                    newItems.push(item)
                }
            }
            this.items = []
            this.items = newItems
            this.selected = []
        },
        rulesFunc() {
            if(this.editAttribute === "FileRoot") {
                const rule1 = value => !!value || 'Required.'
                this.rules.push(rule1)
                var fileroots = []
                for (var i of this.items) {
                    fileroots.push(i['FileRoot'])
                }
                const rule2 = value => (this.itemToEdit['FileRoot'] == value) || (fileroots.indexOf(value) == -1) || 'Duplicate File Root'
                this.rules.push(rule2)
            }
            else {
                this.rules = []
            }
        },
        saveExpectedFiles() {
            this.originalItems = []
            for(var item of this.items) {
                this.originalItems.push(Object.assign({}, item))
            }
            this.myPortco['ExpectedFiles'] = this.originalItems
            return this.myFile
            
        },
        populateItems() {
            this.originalItems = []
            for (var item of this.items) {
                this.originalItems.push(Object.assign({}, item))
            }
            this.forceRenderer()
        },
        changePortcoToEdit(portco) {
            var portcoID = portco['PortcoID']
            //check if it's a new portco
            if (portco['ExpectedFiles'].length == 0) {
                this.myFile.push(Object.assign({}, portco))
            }
            this.myPortco = this.myFile.find(p => p.PortcoID === portcoID)
            this.items = this.myPortco['ExpectedFiles']
            this.populateItems()
        },
        parse() {
            this.myFile = JSON.parse(JSON.stringify(this.portcos))
            for (var portco of this.myFile) {
                var expectedFiles = portco["ExpectedFiles"]
                for (var file of expectedFiles) {
                    file['FileSchema'] = JSON.parse(file['FileSchema'])
                    file['FileSchema'] = file['FileSchema'].join(", ")
                    file['Key_Fields'] = JSON.parse(file['Key_Fields'])
                    file['Key_Fields'] = file['Key_Fields'].join(", ")
                }
            }
            
        },
    },
    mounted() {
        this.parse()
        var index = this.portcos.indexOf(this.portcoToEdit)
        this.myPortco = this.myFile[index]
        this.items = this.myPortco['ExpectedFiles']
        this.populateItems()
    }
}
</script>

<style scoped>
    .v-btn {
        margin: 2em 1em;
    }

    .hide {
        display: none!important;
    }

    .deleteIcon {
        margin-left: 1em;
        margin-top:16px;
        font-size: 30px!important;
    }

    .container {
        margin-top: 64px!important;
    }
</style>
