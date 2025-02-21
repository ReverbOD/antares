<template>
   <div
      ref="tableWrapper"
      class="vscroll"
      :style="{'height': resultsSize+'px'}"
   >
      <TableContext
         v-if="isContext"
         :context-event="contextEvent"
         :selected-rows="selectedRows"
         @delete-selected="deleteSelected"
         @close-context="isContext = false"
      />
      <ul v-if="resultsWithRows.length > 1" class="tab tab-block result-tabs">
         <li
            v-for="(result, index) in resultsWithRows"
            :key="index"
            class="tab-item"
            :class="{'active': resultsetIndex === index}"
            @click="selectResultset(index)"
         >
            <a>{{ result.fields ? result.fields[0].orgTable : '' }} ({{ result.rows.length }})</a>
         </li>
      </ul>
      <div ref="table" class="table table-hover">
         <div class="thead">
            <div class="tr">
               <div
                  v-for="(field, index) in fields"
                  :key="index"
                  class="th c-hand"
               >
                  <div ref="columnResize" class="column-resizable">
                     <div class="table-column-title" @click="sort(field.name)">
                        <i
                           v-if="field.key"
                           class="mdi mdi-key column-key c-help"
                           :class="`key-${field.key}`"
                           :title="keyName(field.key)"
                        />
                        <span>{{ field.alias || field.name }}</span>
                        <i
                           v-if="currentSort === field.name"
                           class="mdi sort-icon"
                           :class="currentSortDir === 'asc' ? 'mdi-sort-ascending':'mdi-sort-descending'"
                        />
                     </div>
                  </div>
               </div>
            </div>
         </div>
         <BaseVirtualScroll
            v-if="resultsWithRows[resultsetIndex] && resultsWithRows[resultsetIndex].rows"
            ref="resultTable"
            :items="sortedResults"
            :item-height="22"
            class="tbody"
            :visible-height="resultsSize"
            :scroll-element="scrollElement"
         >
            <template slot-scope="{ items }">
               <WorkspaceQueryTableRow
                  v-for="row in items"
                  :key="row._id"
                  :row="row"
                  :fields="fields"
                  :key-usage="keyUsage"
                  class="tr"
                  :class="{'selected': selectedRows.includes(row._id)}"
                  @select-row="selectRow($event, row._id)"
                  @update-field="updateField($event, getPrimaryValue(row))"
                  @contextmenu="contextMenu"
               />
            </template>
         </basevirtualscroll>
      </div>
   </div>
</template>

<script>
import { uidGen } from 'common/libs/uidGen';
import BaseVirtualScroll from '@/components/BaseVirtualScroll';
import WorkspaceQueryTableRow from '@/components/WorkspaceQueryTableRow';
import TableContext from '@/components/WorkspaceQueryTableContext';
import { mapActions, mapGetters } from 'vuex';

export default {
   name: 'WorkspaceQueryTable',
   components: {
      BaseVirtualScroll,
      WorkspaceQueryTableRow,
      TableContext
   },
   props: {
      results: Array,
      tabUid: [String, Number]
   },
   data () {
      return {
         resultsSize: 1000,
         localResults: [],
         isContext: false,
         contextEvent: null,
         selectedCell: null,
         selectedRows: [],
         currentSort: '',
         currentSortDir: 'asc',
         resultsetIndex: 0,
         scrollElement: null
      };
   },
   computed: {
      ...mapGetters({
         getWorkspaceTab: 'workspaces/getWorkspaceTab'
      }),
      primaryField () {
         return this.fields.filter(field => ['pri', 'uni'].includes(field.key))[0] || false;
      },
      sortedResults () {
         if (this.currentSort) {
            return [...this.localResults].sort((a, b) => {
               let modifier = 1;
               const valA = typeof a[this.currentSort] === 'string' ? a[this.currentSort].toLowerCase() : a[this.currentSort];
               const valB = typeof b[this.currentSort] === 'string' ? b[this.currentSort].toLowerCase() : b[this.currentSort];
               if (this.currentSortDir === 'desc') modifier = -1;
               if (valA < valB) return -1 * modifier;
               if (valA > valB) return 1 * modifier;
               return 0;
            });
         }
         else
            return this.localResults;
      },
      resultsWithRows () {
         return this.results.filter(result => result.rows);
      },
      fields () {
         return this.getWorkspaceTab(this.tabUid) && this.getWorkspaceTab(this.tabUid).fields[this.resultsetIndex] ? this.getWorkspaceTab(this.tabUid).fields[this.resultsetIndex] : [];
      },
      keyUsage () {
         return this.getWorkspaceTab(this.tabUid) && this.getWorkspaceTab(this.tabUid).keyUsage[this.resultsetIndex] ? this.getWorkspaceTab(this.tabUid).keyUsage[this.resultsetIndex] : [];
      }
   },
   watch: {
      results () {
         this.setLocalResults();
         this.resultsetIndex = 0;
      },
      resultsetIndex () {
         this.setLocalResults();
      }
   },
   updated () {
      if (this.$refs.table)
         this.refreshScroller();

      if (this.$refs.tableWrapper)
         this.scrollElement = this.$refs.tableWrapper;
   },
   mounted () {
      window.addEventListener('resize', this.resizeResults);
   },
   destroyed () {
      window.removeEventListener('resize', this.resizeResults);
   },
   methods: {
      ...mapActions({
         addNotification: 'notifications/addNotification'
      }),
      fieldType (cKey) {
         let type = 'unknown';
         const field = this.fields.filter(field => field.name === cKey)[0];
         if (field)
            type = field.type;

         return type;
      },
      fieldPrecision (cKey) {
         let length = 0;
         const field = this.fields.filter(field => field.name === cKey)[0];
         if (field)
            length = field.datePrecision;

         return length;
      },
      keyName (key) {
         switch (key) {
            case 'pri':
               return 'PRIMARY';
            case 'uni':
               return 'UNIQUE';
            case 'mul':
               return 'INDEX';
            default:
               return 'UNKNOWN ' + key;
         }
      },
      getTable (index) {
         if (this.resultsWithRows[index] && this.resultsWithRows[index].fields && this.resultsWithRows[index].fields.length)
            return this.resultsWithRows[index].fields[0].orgTable;
         return '';
      },
      getPrimaryValue (row) {
         const primaryFieldName = Object.keys(row).find(prop => [
            this.primaryField.alias,
            this.primaryField.name,
            `${this.primaryField.table}.${this.primaryField.alias}`,
            `${this.primaryField.table}.${this.primaryField.name}`
         ].includes(prop));
         return row[primaryFieldName];
      },
      setLocalResults () {
         this.resetSort();
         this.localResults = this.resultsWithRows[this.resultsetIndex] && this.resultsWithRows[this.resultsetIndex].rows ? this.resultsWithRows[this.resultsetIndex].rows.map(item => {
            return { ...item, _id: uidGen() };
         }) : [];
      },
      resizeResults () {
         if (this.$refs.resultTable) {
            const el = this.$refs.tableWrapper;

            if (el) {
               const footer = document.getElementById('footer');
               const size = window.innerHeight - el.getBoundingClientRect().top - footer.offsetHeight;
               this.resultsSize = size;
            }
            this.$refs.resultTable.updateWindow();
         }
      },
      refreshScroller () {
         this.resizeResults();
      },
      updateField (payload, id) {
         if (!this.primaryField)
            this.addNotification({ status: 'warning', message: this.$t('message.unableEditFieldWithoutPrimary') });
         else {
            const params = {
               primary: this.primaryField.name,
               table: this.getTable(this.resultsetIndex),
               id,
               ...payload
            };
            this.$emit('update-field', params);
         }
      },
      deleteSelected () {
         if (!this.primaryField)
            this.addNotification({ status: 'warning', message: this.$t('message.unableEditFieldWithoutPrimary') });
         else {
            const rowIDs = this.localResults.filter(row => this.selectedRows.includes(row._id)).map(row => row[this.primaryField.name]);
            const params = {
               primary: this.primaryField.name,
               table: this.getTable(this.resultsetIndex),
               rows: rowIDs
            };
            this.$emit('delete-selected', params);
         }
      },
      applyUpdate (params) {
         const { primary, id, field, table, content } = params;

         this.localResults = this.localResults.map(row => {
            if (row[primary] === id)// only fieldName
               row[field] = content;
            else if (row[`${table}.${primary}`] === id)// table.fieldName
               row[`${table}.${field}`] = content;

            return row;
         });
      },
      selectRow (event, row) {
         if (event.ctrlKey) {
            if (this.selectedRows.includes(row))
               this.selectedRows = this.selectedRows.filter(el => el !== row);
            else
               this.selectedRows.push(row);
         }
         else if (event.shiftKey) {
            if (!this.selectedRows.length)
               this.selectedRows.push(row);
            else {
               const lastID = this.selectedRows.slice(-1)[0];
               const lastIndex = this.localResults.findIndex(el => el._id === lastID);
               const clickedIndex = this.localResults.findIndex(el => el._id === row);
               if (lastIndex > clickedIndex) {
                  for (let i = clickedIndex; i < lastIndex; i++)
                     this.selectedRows.push(this.localResults[i]._id);
               }
               else if (lastIndex < clickedIndex) {
                  for (let i = clickedIndex; i > lastIndex; i--)
                     this.selectedRows.push(this.localResults[i]._id);
               }
            }
         }
         else
            this.selectedRows = [row];
      },
      contextMenu (event, cell) {
         this.selectedCell = cell;
         if (!this.selectedRows.includes(cell.id))
            this.selectedRows = [cell.id];
         this.contextEvent = event;
         this.isContext = true;
      },
      sort (field) {
         if (field === this.currentSort) {
            if (this.currentSortDir === 'asc')
               this.currentSortDir = 'desc';
            else
               this.resetSort();
         }
         else {
            this.currentSortDir = 'asc';
            this.currentSort = field;
         }
      },
      resetSort () {
         this.currentSort = '';
         this.currentSortDir = 'asc';
      },
      selectResultset (index) {
         this.resultsetIndex = index;
      }
   }
};
</script>

<style lang="scss" scoped>
.vscroll {
  height: 1000px;
  overflow: auto;
  overflow-anchor: none;
}

.column-resizable {
  &:hover,
  &:active {
    resize: horizontal;
    overflow: hidden;
  }
}

.table-column-title {
  display: flex;
  align-items: center;
}

.sort-icon {
  font-size: 0.7rem;
  line-height: 1;
  margin-left: 0.2rem;
}

.result-tabs {
  background: transparent !important;
  margin: 0;
}
</style>
