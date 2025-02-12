<template lang="pug">
div.vue-editable-grid
  .grid-tools
    span.infolines ligne {{ actualRow }}, colonne {{ actualCol }}
    span.infolines Nombre de lignes sur le fichier : {{ nbLinesFile }}
  .grid-table-container(ref='container')
    table.grid-table(style="border-collapse: collapse;" ref='table' )
      thead(ref='head')
        tr.headers-row(:style='{ "grid-template-columns": gridTemplateColumns }')
          th(
            v-for='(column, index) in columnDefs'
            :key='index'
            :class='{ sortable: column.sortable, sorting: column.field === sortByColumn, descending: sortByDesc }'
            @click='sort(column)'
            style="border: 1px solid #ebebeb; border-collapse: collapse;"
          )
            span.header-content
              span.header-name
                span(v-if='!column.rename' :ref="column.headerName") {{ column.headerName }} 
                span(v-if = 'column.rename')
                  input(
                      type='text'
                      size="20"
                      v-model='newNameHeader'
                    )
                  span &nbsp;&nbsp;
                  a(@click='renameField(column.headerName)')
                    img(src="../static/images/tick.png", width="15")
                span( v-if='!column.optional && index != 0') *
              span.intermediary
              span(v-if='index != 0' class="menuIcon" @click="moveHeaderMenu(column.headerName)") 
                img(src="../static/images/menu.png", width="15")
            span.resize-handle(@mousedown='initResize(column, $event)' @click.stop)
      tbody(ref='body', id='body', v-on="handleScroll()")
        div(:style=' { "min-height": `${rowDataPage.length * itemHeight}px` }')
          tr.gridrow(
            v-for='(row, rowIndex) in visibleRows' 
            :key='row[rowDataKey]' 
            :style='{ "grid-template-columns": gridTemplateColumns, transform: `translateY(${(itemHeight * rowIndex) + ((itemHeight * offsetRows))}px)`, height: `${itemHeight}px`, width: `100%` }'
          )
            cell(
              v-for='(column, columnIndex) in columnDefs'
              v-bind:style='rowDataColor[rowIndex][fieldNames[columnIndex-1]] ? {"border": `2px solid ${rowDataColor[rowIndex][fieldNames[columnIndex-1]]}`, "border-collapse": "collapse" } : {}'
              :ref='`cell${rowIndex}-${columnIndex}`'
              :key='columnIndex'
              :column='column'
              :row='row'
              :columnIndex='columnIndex'
              :rowIndex='offsetRows + rowIndex'
              :selStart='selStart'
              :selEnd='selEnd'
              :cellEditing='cellEditing'
              :cellsWithErrors='cellsWithErrors'
              :onlyBorder='onlyBorder'
              @click='selectCell(offsetRows + rowIndex, columnIndex, $event)'
              @dblclick='tryEdit(row, column, offsetRows + rowIndex, columnIndex)'
              @edited='cellEdited'
              @edit-cancelled='cellEditing = []'
              @link-clicked='linkClicked(row, column, offsetRows + rowIndex, columnIndex)'
              @contextmenu='contextMenu(row, column, rowIndex, columnIndex, $event)'
              @mousedown='startSelection(offsetRows + rowIndex, columnIndex)'
              @mouseover='onSelection(offsetRows + rowIndex, columnIndex)'
              @mouseup='stopSelection'
              @show-array-enum='showArrayEnum'
              @show-geopoint='showGeopoint'
              @get-identifier='getIdenfifier'
            )
    textarea.hidde(ref='tmp')
</template>

<script>
/* eslint no-use-before-define: 0 */


import filterAndSort from './filter-and-sort';
import { checkFocus, cellValueParser } from './helpers';
import { cellFormatter } from './vue-filters';
import { initResize } from './header-resize';
import Paginate from './Paginate.vue';
import Cell from './Cell.vue';
import Filters from './Filters.vue';

const isWriteableKey = (code) => (code >= 48 && code <= 57) // 0 - 9
  || (code >= 65 && code <= 90) // a-z
  || (code >= 96 && code <= 105) // 0-9 numpad
  || (code >= 188 && code <= 191); //

let changePending = false;

export default {
  components: { Paginate, Cell, Filters },
  props: {
    id: { type: String, required: true },
    columnDefs: { type: Array, required: true },
    rowData: { type: Array, required: true },
    rowDataColor: { type: Array, required: true },
    fieldNames: { type: Array, required: true },
    rowDataKey: { type: String, required: true },
    enableFilters: { type: Boolean, default: true },
    pageCount: { type: Number, default: 0 },
    itemHeight: { type: Number, default: 40 },
    virtualScrollOffset: { type: Number, default: 3 },
    onlyBorder: { type: Boolean, default: true },
    actualRow: { type: Number, default: 0},
    actualCol: { type: Number, default: 1},
    nbLinesFile: { type: Number, default: 0}
  },
  data() {
    return {
      selStart: [],
      selEnd: [],
      filter: {},
      sortByColumn: null,
      sortByDesc: false,
      columnResizing: null,
      cellEditing: [],
      focused: false,
      page: 0,
      cellsWithErrors: {},
      gridTemplateColumns: null,
      selectedRowKey: null,
      // virtual scroll
      offsetRows: 0,
      visibleRows: [],
      isSelecting: false,
      selStartSelection: [],
      scrollLeft: null,
      newNameHeader: '',
    };
  },
  created() {  

    document.addEventListener('keydown', ($event) => {
      if (!this.focused || this.cellEditing.length) {
        return;
      }
      const key = $event.key;
      const isControl = $event.metaKey || $event.ctrlKey;
      const isShift = $event.shiftKey;
      if (key === 'ArrowDown') {
        if (isControl) this.selectLastRow();
        else this.sumSelectionRow(1);
        $event.preventDefault();
      } else if (key === 'ArrowRight') {
        if (isControl) this.selectLastCol();
        else this.sumSelectionCol(1);
        $event.preventDefault();
      } else if (key === 'ArrowUp') {
        if (isControl) this.selectFirstRow();
        else this.sumSelectionRow(-1);
        $event.preventDefault();
      } else if (key === 'ArrowLeft') {
        // if (isControl) this.selectFirstCol()
        // else
        this.sumSelectionCol(-1);
        $event.preventDefault();
      } else if (key === 'Tab') {
        this.sumSelectionCol(isShift ? -1 : 1);
        $event.preventDefault();
      } else if (key === 'Enter') {
        this.sumSelectionRow(1);
        $event.preventDefault();
      } else if (key === 'Delete') {
        const { colData, rowData, rowIndex, colIndex } = this.getCell();
        this.setEditableValue(rowData, colData, rowIndex, colIndex, null, true, null);
      } else if (key === 'F2') {
        const { colData, rowData, rowIndex, colIndex } = this.getCell();
        this.tryEdit(rowData, colData, rowIndex, colIndex);
      } else if (key.toLowerCase() === 'v' && isControl) {
        this.$refs.tmp.value = '';
        this.$refs.tmp.focus();
        setTimeout(() => {
          const pasted = this.$refs.tmp.value;
          const arrayPasted = pasted.split('\n').filter((row, index, array) => {
            const isLastRow = index === array.length - 1;
            return !(isLastRow && row === '');
          }).map((row) => row.split('\t'));
          const [sRowIndex, sColIndex] = this.selStart;
          const [eRowIndex, eColIndex] = this.selEnd;
          // paste all
          if (sRowIndex === eRowIndex && sColIndex === eColIndex) {
            let rowIndex = null;
            let columnIndex = null;
            let cell = this.getCell();
            if(arrayPasted.length > (this.rowData.length - cell.rowIndex)) {
              this.addMultRows(cell.rowIndex-this.rowData.length+arrayPasted.length);
            }
            arrayPasted.forEach((rowsData, rIdx) => {
              rowsData.forEach((value, cIdx) => {
                const row = this.rowDataPage[sRowIndex + rIdx];
                const column = this.columnDefs[sColIndex + cIdx];
                if (row && column && column.editable) {
                  rowIndex = sRowIndex + rIdx;
                  columnIndex = sColIndex + cIdx;
                  this.setCellError(rowIndex, columnIndex, false);
                  try {
                    // eslint-disable-next-line no-param-reassign
                    value = cellValueParser(column, row, value, false);
                  } catch (error) {
                    this.setCellError(rowIndex, columnIndex, error);
                    // eslint-disable-next-line no-param-reassign
                    value = null;
                  }
                  this.setEditableValue(row, column, rowIndex, columnIndex, value, true, null);
                }
              });
            });
            this.selEnd = [rowIndex, columnIndex];
          }
        }, 100);
      } else if (isControl && (key.toLowerCase() === 'c' || key.toLowerCase() === 'x')) {
        if (isShift) $event.preventDefault();
        this.copyToClipboard(isShift);
      } else if (!$event.metaKey && this.selStart[0] >= 0 && isWriteableKey($event.keyCode)) {
        
        const { colData, rowData, rowIndex, colIndex } = this.getCell();
        $event.preventDefault();
        this.tryEdit(rowData, colData, rowIndex, colIndex, $event.key);
      }
    });
  },
  mounted() {
    checkFocus(this.$refs.container.querySelector('tbody'), (focused) => {
      this.focused = focused;
    });
    this.loadColumnsSizes();
    const body = this.$refs.body;
    body.addEventListener('scroll', (e) => {
      requestAnimationFrame(() => {
        this.$refs.head.scrollLeft = e.target.scrollLeft;
      });
      this.renderVisibleScroll(body);
    });
    this.renderVisibleScroll(body);
  },
  watch: {
    selStart(value, old) {
      this.emitRowSelected();
    },
    selEnd() {
      this.emitRowSelected();
    },
    rowDataPage() {
      this.emitRowSelected();
    },
    rowData() {
      const [rowIndex] = this.selStart;
      const currentKey = this.rowDataPage[rowIndex] && this.rowDataPage[rowIndex][this.rowDataKey];
      if (currentKey !== this.selectedRowKey) {
        this.emitRowSelected();
      }
      this.renderVisibleScroll();
    },
    columnDefs() {
      this.loadColumnsSizes();
    },
  },
  computed: {
    rowDataFiltered() {
      return filterAndSort(
        this.filter,
        this.rowData,
        this.columnDefs,
        this.sortByColumn,
        this.sortByDesc,
      );
    },
    rowDataPage() {
      if (!this.pageCount) {
        return this.rowDataFiltered;
      }
      return this.rowDataFiltered
        .slice(this.page * this.pageCount, this.page * this.pageCount + this.pageCount);
    },
    columnDefsFiltered() {
      return this.columnDefs;
    },
    pages() {
      return Math.ceil(this.rowDataFiltered.length / this.pageCount)
    },
  },
  methods: {
    showGeopoint(row,col, column, val) {
      this.$emit('show-geopoint', row, col, column, val, this.$refs[`cell${row}-${col}`][0].$el.getBoundingClientRect());
    },
    getIdenfifier(row,col, column, val) {
      this.$emit('get-identifier', row, col, column, val, this.$refs[`cell${row}-${col}`][0].$el.getBoundingClientRect());
    },
    showArrayEnum(row,col, column, val){
      this.$emit('show-array-enum', row, col, column, val, this.$refs[`cell${row}-${col}`][0].$el.getBoundingClientRect());
    },
    handleScroll(){
      if(this.$refs.body) {
        if(this.$refs.body.scrollLeft != this.scrollLeft){
          this.scrollLeft = this.$refs.body.scrollLeft;
          this.$emit('remove-boxes');
        }
      }
    },
    addMultRows(val) {
      this.$emit('add-multiple-rows', { rowsToAdd: val });
    },
    addEmptyRow() {
      this.$emit('add-empty-row');
    },
    emitRowSelected() {
      if ((this.selStart[0] === this.selEnd[0] && this.selStart[1] === this.selEnd[1])) {
        const cell = this.getCell();

        this.$emit('row-selected', cell);
      } else {
        this.$emit('row-selected', { rowData: null });
      }
    },
    renderVisibleScroll(body) {
      if (!body) {
        // eslint-disable-next-line no-param-reassign
        body = this.$refs.body;
      }
      const { itemHeight } = this;
      let offset = Math.floor(body.scrollTop / itemHeight);
      if (offset < this.virtualScrollOffset) {
        offset = 0;
      } else {
        offset -= this.virtualScrollOffset;
      }
      const visibleCount = Math.ceil(body.clientHeight / itemHeight) + 1 + this.virtualScrollOffset;
      this.offsetRows = offset;
      this.visibleRows = this.rowDataPage.slice(offset, offset + visibleCount);
    },
    filtersChanged() {
      this.page = 0;
      this.renderVisibleScroll();
    },
    getCell() {
      const [rowIndex, colIndex] = this.selStart;
      return this.getCellByRef(rowIndex, colIndex);
    },
    getCellByRef(rowIndex, colIndex) {
      const colData = this.columnDefs[colIndex];
      const rowData = this.rowDataFiltered[rowIndex];
      return {
        rowData,
        colData,
        rowIndex,
        colIndex,
      };
    },
    loadColumnsSizes() {
      const columns = JSON.parse(localStorage.getItem(`${this.id}_columns`) || '[]');
      this.columnDefs.forEach((column) => {
        const previousSize = columns.find((c) => c.field === column.field);
        // eslint-disable-next-line no-param-reassign
        column.size = (previousSize && previousSize.size) || column.size;
      });
      this.setGridColumnTemplate();
    },
    async selectCell(rowIndex, colIndex, $event) {
      this.$emit('show-errors', $event, rowIndex, colIndex, this.$refs[`cell${rowIndex}-${colIndex}`][0].$el.getBoundingClientRect());

      if (changePending) {
        return;
      }
      if (this.cellEditing.length
        && this.cellEditing[0] === rowIndex
        && this.cellEditing[1] === colIndex) {
        return;
      }
      const maxrow = this.rowDataFiltered.length - 1;
      const maxcol = this.columnDefs.length - 1;
      rowIndex = rowIndex < 0 ? 0 : rowIndex > maxrow ? maxrow : rowIndex;
      colIndex = colIndex < 0 ? 0 : colIndex > maxcol ? maxcol : colIndex;
      const shift = $event && $event.shiftKey;
      if (shift) {
        const rowRange = [this.selStart[0], rowIndex];
        const colRange = [this.selStart[1], colIndex];
        this.selStart = [Math.min(...rowRange), Math.min(...colRange)];
        this.selEnd = [Math.max(...rowRange), Math.max(...colRange)];
      } else {
        this.selEnd = [rowIndex, colIndex];
        this.selStart = [rowIndex, colIndex];
      }
      if (this.cellEditing[0] !== rowIndex || this.cellEditing[1] !== colIndex) {
        this.cellEditing = [];
      }
      this.focus();
    },
    focus() {
      const [rowIndex, colIndex] = this.selStart;
      const body = this.$refs.body;
      const cell = body.querySelector(`#cell${rowIndex}-${colIndex}`);
      if (cell) {
        const cellHPosition = cell.offsetLeft - body.scrollLeft;
        const cellVPosition = cell.getBoundingClientRect().top - body.getBoundingClientRect().top;
        if (cellHPosition < 0) {
          body.scrollLeft += cellHPosition;
        } else {
          const cellRightPosition = cellHPosition + cell.clientWidth;
          if (cellRightPosition > body.clientWidth) {
            body.scrollLeft += cellRightPosition - body.clientWidth + 2;
          }
        }
        if (cellVPosition < 0) {
          body.scrollTop += cellVPosition;
        } else {
          const cellBottomPosition = cellVPosition + cell.clientHeight;
          if (cellBottomPosition > body.clientHeight) {
            body.scrollTop += cellBottomPosition - body.clientHeight + 2;
          }
        }
      } else if (rowIndex === 0) {
        body.scrollTop = 0;
      } else if (rowIndex === this.rowData.length - 1) {
        body.scrollTop = this.rowDataPage.length * this.itemHeight;
      }
    },
    tryEdit (row, column, rowIndex, columnIndex, newValue) {
      if (column.editable) {
        this.cellEditing = [rowIndex, columnIndex, newValue];
      }
    },
    linkClicked(rowData, colData, rowIndex, colIndex, newValue) {
      this.$emit('link-clicked', { rowData, colData, rowIndex, colIndex });
    },
    contextMenu(row, column, rowIndex, columnIndex, $event) {
      this.isSelecting = false;
      const isInside = rowIndex >= this.selStart[0] && rowIndex <= this.selEnd[0] && columnIndex >= this.selStart[1] && columnIndex <= this.selEnd[1];
      if (!isInside) this.selectCell(this.offsetRows + rowIndex, columnIndex, $event);
      this.$emit('context-menu', { row, column, rowIndex, columnIndex, $event });
    },
    setCellError(rowIndex, columnIndex, error) {
      const cellId = `cell${rowIndex}-${columnIndex}`;
      if (error) {
        this.$set(this.cellsWithErrors, cellId, error);
      } else {
        this.$delete(this.cellsWithErrors, cellId);
      }
    },
    cellEdited({ row, column, rowIndex, columnIndex, value, $event, valueChanged }) {
      this.setEditableValue(row, column, rowIndex, columnIndex, value, valueChanged, $event);
    },
    setEditableValue(row, column, rowIndex, columnIndex, value, valueChanged, $event) {
      return new Promise((resolve) => {
        if (!valueChanged) {
          this.cellEditing = [];
          resolve();
          return;
        }
        const cellId = `cell${rowIndex}-${columnIndex}`;
        const input = document.querySelector(`#${cellId} input`);
        const eventCode = $event && $event.code;
        let prevent = false;

        const preventDefault = () => {
          prevent = true;
          if (changePending) {
            changePending = false;
            this.cellEditing = [];
          }
        };
        const markAsPending = () => {
          changePending = true;
          if (input) {
            input.disabled = true;
          }
        };
        const confirm = () => {
          if (prevent) return;
          changePending = false;
          // eslint-disable-next-line no-param-reassign
          row[column.field] = value;
          this.cellEditing = [];
          if (eventCode === 'Enter') {
            this.sumSelectionRow(1);
          }
          resolve();
        };
        const markAsFailed = (error) => {
          this.setCellError(rowIndex, columnIndex, error || 'Error in value or value format');
        };
        const markAsSuccess = () => {
          this.setCellError(rowIndex, columnIndex, false);
        };

        this.$emit('cell-updated', { value, row, column, rowIndex, columnIndex, $event, preventDefault, markAsPending, confirm, markAsFailed, markAsSuccess });
        if (prevent) {
          this.cellEditing = [];
          resolve();
        } else if (!changePending) {
          confirm();
        }

        if (eventCode === 'Tab') {
          $event.preventDefault();
        }
      });
    },
    sort(column) {
      if (column.sortable) {
        if (this.sortByColumn === column.field) {
          this.sortByDesc = !this.sortByDesc;
        }
        this.sortByColumn = column.field;
        this.renderVisibleScroll();
      }
    },
    setGridColumnTemplate() {
      this.gridTemplateColumns = this.columnDefsFiltered
        .map(({ size }) => size || 'minmax(130px, 1.67fr)')
        .join(' ');
    },
    initResize(column, $event) {
      initResize($event.target.parentNode, $event, width => {
        column.size = Math.max(30, width) + 'px';
        this.setGridColumnTemplate();
      }, () => {
        localStorage.setItem(`${this.idRowVEG}_columns`, JSON.stringify(this.columnDefsFiltered.map(({ field, size }) => ({ field, size }))));
      });
    },
    selectFirstCol() {
      this.selectCell(this.selStart[1], 1);
    },
    selectLastCol() {
      this.selectCell(this.selStart[1], this.columnDefs.length - 1);
    },
    selectFirstRow() {
      this.selectCell(0, this.selStart[1]);
    },
    selectLastRow() {
      this.selectCell(this.rowData.length - 1, this.selStart[1]);
    },
    sumSelectionCol(sum) {
      let [row, col] = this.selStart;
      col += sum;
      if (col !== 0) {
        this.selectCell(row, col);
      }
    },
    sumSelectionRow(sum) {
      if(sum == '1'){
        const cell = this.getCell();
        this.$emit('maybe-add-row', cell);
      }

      let [row, col] = this.selStart;
      row += sum;
      this.selectCell(row, col);
    },
    removeFilter(field) {
      if (field) {
        this.$delete(this.filter, field);
      } else {
        this.filter = {};
      }
      this.renderVisibleScroll();
    },
    getFormattedRows() {
      return this.rowData.map((row) => {
        return this.columnDefs.reduce((rowFormatted, column) => {
          rowFormatted[column.field] = cellFormatter(row[column.field], column)
          return rowFormatted;
        }, {})
      });
    },
    startSelection (rowIndex, colIndex) {
      this.isSelecting = true
      this.selStartSelection = [rowIndex, colIndex]
    },
    onSelection (rowIndex, colIndex) {
      if (this.isSelecting) {
        if (rowIndex < this.selStartSelection[0]) {
          this.selStart = colIndex < this.selStartSelection[1] ? [rowIndex, colIndex] : [rowIndex, this.selStartSelection[1]];
          this.selEnd = colIndex < this.selStartSelection[1] ? this.selStartSelection : [this.selStartSelection[0], colIndex];
        } else {
          this.selStart = colIndex < this.selStartSelection[1] ? [this.selStartSelection[0], colIndex] : this.selStartSelection;
          this.selEnd = colIndex < this.selStartSelection[1] ? [rowIndex, this.selStartSelection[1]] : [rowIndex, colIndex];
        }
      }
    },
    stopSelection() {
      this.isSelecting = false;
    },
    copyToClipboard(withHeaders) {
      const [rowStart, columnStart] = this.selStart;
      const [rowEnd, columnEnd] = this.selEnd;
      let value = withHeaders ? this.columnDefs.filter((_, index) => index >= columnStart && index <= columnEnd).map(col => col.headerName).join('\t') + '\n' : '';
      for (let row = rowStart; row <= rowEnd; row++) {
        for (let column = columnStart; column <= columnEnd; column++) {
          const { colData, rowData } = this.getCellByRef(row, column);
          const cellValue = cellFormatter(rowData[colData.field], colData, rowData);
          value = column === columnEnd ? `${value}${cellValue}\n` : `${value}${cellValue}\t`;
        }
      }
      this.$refs.tmp.value = value || '';
      this.$refs.tmp.select();
      document.execCommand('copy');
    },
    columnOperation(event,column){
      this.$emit('column-operation', event.target.value, column);
    },
    moveHeaderMenu(column){
      this.$emit('moveHeaderMenu', this.$refs[column][0].getBoundingClientRect(), column);
    },
    renameField(oldname) {
      this.$emit('rename-field', this.newNameHeader, oldname);
      this.newNameHeader = '';
    }
  },
};
</script>

<style lang="scss" scoped>
@import './variables';

$tools-height: 25px;

.scroller {
  height: 100%;
}

.grid-tools,
.grid-tools-left,
.grid-tools-right {
  height: $tools-height;
  display: flex;
  align-items: center;
}

.grid-tools {
  justify-content: space-between;
}

.grid-tools-right {
  justify-items: flex-end;
}

.grid-tool {
  margin-right: 10px;
}

.grid-table-container {
  height: calc(100% - #{$tools-height});
}

.grid-table {
  height: 100%;
  display: grid;
  grid-template-rows: 35px auto;
  overflow: hidden;

  .headers-row {
    height: 100%;
  }

  &.filters {
    grid-template-rows: 70px auto;

    .headers-row,
    .filters-row {
      height: 50%;
    }
  }
}

thead,
tbody {
  display: block;
  border-collapse: collapse;
  min-width: 100%;
}

thead {
  overflow-x: hidden;
  box-shadow: 0px 1px 5px #cecece;
}

tbody {
  overflow-y: scroll;
  position: relative;
}

thead.resizing th {
  cursor: col-resize;
}

tr {
  display: grid;
  height: 100%;
}

.gridrow {
  position: absolute;
  top: 0;
  min-width: 100%;
}

th {
  padding: 0 $cell-side-paddings;
  display: flex;
  align-items: center;

  .header-content {
    text-overflow: ellipsis;
    white-space: nowrap;
    display: flex;
    overflow: hidden;
    width: 100%;
  }
}

th {
  position: sticky;
  top: 0;
  border-bottom: solid 1px #ebebeb;
  text-align: left;
  font-weight: normal;
  position: relative;

  &::after {
    margin-left: 5px;
  }

  &.sortable {
    cursor: pointer;
    border-right: 1px solid #ebebeb;
    border-top: 1px solid #ebebeb;
    background-color: #fcfcfc;
    &::after {
      position: absolute;
      right: 0;
    }

    &:not(.sorting) {
      &::after {
        content: '↑↓';
        opacity: 0;
        transition: opacity 0.3s;
      }

      &:hover::after {
        opacity: 0.5;
      }
    }
  }

  &.sorting::after {
    content: '↑';
  }

  &.sorting.descending::after {
    content: '↓';
  }

  &.filter {
    display: flex;
    padding: 0;

    input {
      width: 100%;
      border: none;
      height: 30px;
      padding: 0 5px 0 20px;
      background: transparent;
      z-index: 1;
    }

    &::after {
      background-image: url('data:image/svg+xml;base64,PD94bWwgdmVyc2lvbj0iMS4wIiA/PjxzdmcgaGVpZ2h0PSIxNzkyIiB2aWV3Qm94PSIwIDAgMTc5MiAxNzkyIiB3aWR0aD0iMTc5MiIgeG1sbnM9Imh0dHA6Ly93d3cudzMub3JnLzIwMDAvc3ZnIj48cGF0aCBkPSJNMTU5NSAyOTVxMTcgNDEtMTQgNzBsLTQ5MyA0OTN2NzQycTAgNDItMzkgNTktMTMgNS0yNSA1LTI3IDAtNDUtMTlsLTI1Ni0yNTZxLTE5LTE5LTE5LTQ1di00ODZsLTQ5My00OTNxLTMxLTI5LTE0LTcwIDE3LTM5IDU5LTM5aDEyODBxNDIgMCA1OSAzOXoiLz48L3N2Zz4=');
      background-size: 15px 15px;
      background-repeat: no-repeat;
      background-position: center;
      content: '';
      height: 30px;
      width: 20px;
      display: block;
      position: absolute;
      opacity: 0.3;
      margin: 0;
    }
  }
}

.resize-handle {
  position: absolute;
  top: 0;
  right: 0;
  bottom: 0;
  // background: black;
  opacity: 0;
  width: 10px;
  cursor: col-resize;
  z-index: 1;
}

.resize-handle:hover,
/* The following selector is needed so the handle is visible during
resize even if the mouse isn't over the handle anymore */
.header--being-resized .resize-handle {
  opacity: 0.5;
}

th:hover .resize-handle {
  opacity: 0.3;
}

.hidde {
  position: absolute;
  left: -9999px;
  top: 0;
}

.btnAdd {
  color: #ffffff;
  text-align: left;
  background-color: #f7f5f5;
  width: 150px;
  border: 0px;
  text-align: center;
  color: #555555;
}
.btnAdd:hover {
  background-color: #bbbbbb;
  color: white;
}
</style>
