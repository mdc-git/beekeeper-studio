<template>
  <div class="result-table">
    <div ref="tabulator"></div>
    <statusbar class="tabulator-footer">
      <div class="col x12 flex flex-center">
        <span
          ref="paginationArea"
          class="tabulator-paginator"
          v-show="this.meta.count > this.limit"
        ></span>
      </div>
    </statusbar>
  </div>
</template>

<script type="text/javascript">
import Tabulator from "tabulator-tables";
import Converter from "../../mixins/data_converter";
import Mutators from "../../mixins/data_mutators";
import Statusbar from "../common/StatusBar";
import _ from "lodash";
import dateFormat from "dateformat";
import Papa from "papaparse";
import FileSaver from "file-saver";

export default {
  components: { Statusbar },
  mixins: [Converter, Mutators],
  data() {
    return {
      tabulator: null,
      limit: 1000,
      togglesort: "asc",
      orderBy: "",
    };
  },
  props: ["result", "tableHeight", "query", "active", "connection", "meta"],
  watch: {
    active() {
      if (!this.tabulator) return;
      if (this.active) {
        this.tabulator.restoreRedraw();
        this.$nextTick(() => {
          this.tabulator.redraw();
        });
      } else {
        this.tabulator.blockRedraw();
      }
    },
    tableData: {
      handler() {
        // this.tabulator.replaceData(this.tableData)
      },
    },
    tableColumns: {
      handler() {
        //this.tabulator.setColumns(this.tableColumns);
      },
    },
    tableHeight() {
      this.tabulator.setHeight(this.actualTableHeight);
    },
  },
  computed: {
    actualTableHeight() {
      return "100%";
    },
  },
  beforeDestroy() {
    if (this.tabulator) {
      this.tabulator.destroy();
    }
  },
  async mounted() {
    this.tabulator = new Tabulator(this.$refs.tabulator, {
      ajaxURL: "http://fake",
      ajaxSorting: true,
      ajaxFiltering: true,
      pagination: "remote",
      paginationSize: this.limit,
      paginationElement: this.$refs.paginationArea,
      ajaxRequestFunc: this.dataFetch,
      reactiveData: false,
      virtualDomHoz: true,
      height: this.actualTableHeight,
      nestedFieldSeparator: false,
      cellClick: this.cellClick,
      clipboard: true,
      keybindings: {
        copyToClipboard: false,
      },
      downloadConfig: {
        columnHeaders: true,
      },
    });
  },
  methods: {
    dataFetch(url, config, params) {

      const records_per_page = this.limit
      const records = this.meta.count
      const last_page = Math.ceil(records/records_per_page)
      const page = params?.page ?? 1

      let limit, offset

      let orderBy, orderBy2;

      if (this.meta.orderby) {
        orderBy = ` ORDER BY ${this.meta.orderby} `;
        this.orderBy = orderBy;
      } else {
        orderBy = "";
      }

      if (params.sorters[0]) {
        if (this.page === page || !this.page) {
          this.togglesort = this.togglesort === "asc" ? "desc" : "asc";
        }
        this.page = page;
        orderBy2 = ` ORDER BY ${params.sorters[0].field} ${this.togglesort} `;
      } else {
        orderBy2 = "";
      }

      if (this.togglesort === "asc") {
        offset = (page-1) * records_per_page
        limit = page === last_page ? records - offset :  records_per_page
      } else {
        offset = page === last_page ? 0 : records - page * records_per_page
        limit = page === last_page ? records - (page-1) * records_per_page :  records_per_page
      }

      offset += this.meta.offset

      const result = new Promise((resolve, reject) => {
        (async () => {
          try {
            let sql = `${this.meta.basesql} ${orderBy} LIMIT ${limit} OFFSET ${offset}`;

            if (orderBy2 !== "") {
              sql = `${this.meta.basesql} ${orderBy} LIMIT ${limit} OFFSET ${offset}`;
              sql = `SELECT * FROM ( ${sql} ) beekeeper_sort ${orderBy2}`;
            }
            console.log("->>>>", sql);
            const query = this.connection.query(sql);
            const response = await query.execute();
            Object.freeze(response);
            const fields = response[0].fields;
            const columnWidth = 125;
            const columns = fields.map((column) => {
              const result = {
                title: column.name,
                field: column.name,
                width: columnWidth,
                mutatorData: this.resolveDataMutator(
                  response[0].rows[0][column.name]
                ),
                formatter: this.cellFormatter,
              };
              return result;
            });

            this.tabulator.setColumns(columns);
            const data = await this.dataToTableData(response[0], columns);
            resolve({
              last_page: last_page,
              data,
            });
          } catch (error) {
            reject();
            throw error;
          }
        })();
      });
      return result;
    },
    cellClick(e, cell) {
      this.selectChildren(cell.getElement());
    },
    async download() {
      this.tabulator.modules.ajax.showLoader();
      this.tabulator.blockRedraw();
      
      const response = await this.connection.query(this.query.text).execute();
      Object.freeze(response);

      const dataString = Papa.unparse(response[0].rows);
      Object.freeze(dataString);
      const dateString = dateFormat(new Date(), "yyyy-mm-dd_hMMss");
      const title = this.query.title
        ? _.snakeCase(this.query.title)
        : "query_results";
      FileSaver.saveAs(
        new Blob([dataString], { type: "text/csv;charset=utf-8" }),
        `${title}-${dateString}.csv`
      );

      this.tabulator.modules.ajax.hideLoader();
      this.tabulator.restoreRedraw();
    },
    clipboard() {
      this.tabulator.copyToClipboard("table", true);
      this.$noty.info("Table data copied to clipboard");
    },
  },
};
</script>
