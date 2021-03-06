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
import fs from "fs";
const { app, dialog } = require("electron").remote;
const mysql = require("mysql2");
const stream = require("stream");

export default {
  components: { Statusbar },
  mixins: [Converter, Mutators],
  data() {
    return {
      tabulator: null,
      limit: 1000,
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
    const tabulator = new Tabulator(this.$refs.tabulator, {
      ajaxURL: "http://fake",
      ajaxSorting: true,
      ajaxFiltering: true,
      pagination: "remote",
      paginationSize: this.limit,
      paginationElement: this.$refs.paginationArea,
      ajaxRequestFunc: this.dataFetch,
      reactiveData: false,
      //virtualDomHoz: true,
      height: this.actualTableHeight,
      nestedFieldSeparator: false,
      cellClick: this.cellClick,
      rowClick: this.rowClick,
      clipboard: true,
      keybindings: {
        copyToClipboard: false,
      },
      downloadConfig: {
        columnHeaders: true,
      },
    });
    Object.freeze(tabulator);
    this.tabulator = tabulator;
  },
  methods: {
    dataFetch(url, config, params) {
      // page size
      const records_per_page = this.limit;
      // total result count
      const records = this.meta.count;
      // last page
      const last_page = Math.ceil(records / records_per_page);
      // current page
      const page = params?.page ?? 1;

      let limit = Math.min(
        records - (page - 1) * records_per_page,
        records_per_page
      );

      let offset = (page - 1) * records_per_page;

      let orderBy2 = "";

      console.log(params.sorters);
      // column sorting
      if (params.sorters[0]) {
        if (params.sorters[0].dir === "asc") {
          orderBy2 = `- NULLIF(${params.sorters[0].field},'') desc, ${params.sorters[0].field}`;
        } else {
          orderBy2 = `${params.sorters[0].field} ${params.sorters[0].dir} `;
        }
        
      } else {
        if (this.meta.orderby) {
          orderBy2 = `${this.meta.orderby[1]}`;
        }
      }

      const result = new Promise((resolve, reject) => {
        (async () => {
          try {
            const queryStartTime = +new Date();

            let querytext = this.query.text;
            if (this.meta.orderby) {
              querytext = this.meta.stripped;
            }
            if (orderBy2 !== "") {
              orderBy2 = `ORDER BY ${orderBy2}`;
            }

            let sql = `${querytext} ${orderBy2} LIMIT ${limit} OFFSET ${offset}`;
            if (this.meta.limit) {
              sql = `SELECT * FROM ( ${querytext} ) beekeper_limit ${orderBy2} LIMIT ${limit} OFFSET ${offset}`;
            }

            console.log(sql);

            const query = this.connection.query(sql);
            const response = await query.execute();
            Object.freeze(response);
            const queryEndTime = +new Date();
            const executeTime = queryEndTime - queryStartTime;
            this.$emit("executeTimeUpdate", executeTime);
            if (!this.tabulator.getColumns().length) {
              const fields = response[0].fields;
              const columnWidth = 50;
              const columns = fields.map((column) => {
                const result = {
                  title: column.name,
                  field: column.name,
                  minWidth: columnWidth,
                  mutatorData: this.resolveDataMutator(
                    response[0].rows[0][column.name]
                  ),
                  formatter: this.cellFormatter,
                };
                return result;
              });
              Object.freeze(columns);
              this.tabulator.setColumns(columns);
            }

            const data = await this.dataToTableData(
              response[0],
              this.tabulator.columnManager.columns
            );
            Object.freeze(data);
            resolve({
              last_page: last_page,
              data,
            });
          } catch (error) {
            reject();
            this.$emit("setError", error);
          }
        })();
      });
      return result;
    },
    rowClick: function (e, row) {
      const data = row._row.data;
      const obj = {};
      Object.keys(data).forEach((key) => {
        obj[key] = data[key];
      });
      this.$parent.$parent.$emit("selectedRow", obj);
    },
    cellClick(e, cell) {
      this.selectChildren(cell.getElement());
    },
    transformRow(row) {
      let item = [];
      Object.values(row).forEach((col) => {
        switch (typeof col) {
          case "object":
            col = JSON.stringify(col);
            break;
          case "undefined":
          case "null":
            col = null;
            break;
        }
        item.push('"' + String(col).split('"').join('""') + '"');
      });
      let values = item.join(",");
      values = values + "\n";
      return values;
    },
    async download(format) {
      const dateString = dateFormat(new Date(), "yyyy-mm-dd_hMMss");
      const title = this.query.title
        ? _.snakeCase(this.query.title)
        : "query_results";
      const options = {
        title: "Save file",
        defaultPath:
          app.getPath("downloads") + `/${title}-${dateString}.${format}`,
        buttonLabel: "Save",

        filters: [
          { name: "csv", extensions: ["csv"] },
          { name: "All Files", extensions: ["*"] },
        ],
      };

      const file = await dialog.showSaveDialog(options);
      if (file.filePath === "") return;

      const writer = fs
        .createWriteStream(file.filePath)
        .on("error", function (err) {
          throw err;
        });

      const connection = mysql.createConnection({
        host: this.connection.server.config.host,
        user: this.connection.server.config.user,
        password: this.connection.server.config.password,
        port: this.connection.server.config.port,
        database: this.connection.database.database,
      });
      connection.connect(function (err) {
        if (err) throw err;
      });

      const transformRow = this.transformRow;
      let headerWritten = false;
      const csvTransform = new stream.Transform({
        objectMode: true,
        transform: function (row, encoding, callback) {
          // Do something with the row of data
          if (!headerWritten) {
            writer.write(transformRow(Object.keys(row)));
            headerWritten = true;
          }

          writer.write(transformRow(row));
          callback();
        },
      });

      connection
        .query(this.query.text)
        .stream()
        .pipe(csvTransform)
        .on("finish", function () {
          connection.end();
          writer.close();
        });

      const noty = this.$noty;
      writer.on("finish", function () {
        noty.info("Download finished.<br>Saved file to:<br>" + file.filePath);
      });
    },
    clipboard() {
      this.tabulator.copyToClipboard("table", true);
      this.$noty.info("Table data copied to clipboard");
    },
  },
};
</script>
