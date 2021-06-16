<template>
  <div class="app-container">
    <div class="filter-container">
      <el-input
        v-model="listQuery.title"
        placeholder="Titulo"
        style="width: 200px"
        class="filter-item"
        @keyup.enter.native="handleFilter"
      />
      <el-select
        v-model="listQuery.importance"
        placeholder="Importancia"
        clearable
        style="width: 90px"
        class="filter-item"
      >
        <el-option
          v-for="item in importanceOptions"
          :key="item"
          :label="item"
          :value="item"
        />
      </el-select>
      <el-select
        v-model="listQuery.type"
        placeholder="Tipo"
        clearable
        class="filter-item"
        style="width: 130px"
      >
        <el-option
          v-for="item in calendarTypeOptions"
          :key="item.key"
          :label="item.display_name + '(' + item.key + ')'"
          :value="item.key"
        />
      </el-select>
      <el-select
        v-model="listQuery.sort"
        style="width: 140px"
        class="filter-item"
        @change="handleFilter"
      >
        <el-option
          v-for="item in sortOptions"
          :key="item.key"
          :label="item.label"
          :value="item.key"
        />
      </el-select>
      <el-button
        v-waves
        class="filter-item"
        type="primary"
        icon="el-icon-search"
        @click="handleFilter"
      >
        Buscar
      </el-button>
      <el-button
        class="filter-item"
        style="margin-left: 10px"
        type="primary"
        icon="el-icon-edit"
        @click="handleCreate"
      >
        Añadir
      </el-button>
      <el-button
        v-waves
        :loading="downloadLoading"
        class="filter-item"
        type="primary"
        icon="el-icon-download"
        @click="handleDownload"
      >
        Exportar
      </el-button>
      <el-checkbox
        v-model="showReviewer"
        class="filter-item"
        style="margin-left: 15px"
        @change="tableKey = tableKey + 1"
      >
        Revisor
      </el-checkbox>
    </div>
    <!-- Tabla de Datos -->
    <el-table
      :key="tableKey"
      v-loading="listLoading"
      :data="list"
      border
      fit
      highlight-current-row
      style="width: 100%"
      @sort-change="sortChange"
    >
      <el-table-column
        label="ID"
        prop="id"
        sortable="custom"
        align="center"
        width="80"
        :class-name="getSortClass('id')"
      >
        <template slot-scope="{ row }">
          <span>{{ row.id }}</span>
        </template>
      </el-table-column>

      <el-table-column label="Titulo" min-width="150px">
        <template slot-scope="{ row }">
          <span class="link-type" @click="handleUpdate(row)">{{
            row.title
          }}</span>
          <el-tag>{{ row.type | typeFilter }}</el-tag>
        </template>
      </el-table-column>

      <el-table-column
        v-if="showReviewer"
        label="Revisor"
        width="110px"
        align="center"
      >
        <template slot-scope="{ row }">
          <span style="color: red">{{ row.reviewer }}</span>
        </template>
      </el-table-column>

      <el-table-column label="Estado" class-name="status-col" width="100">
        <template slot-scope="{ row }">
          <el-tag :type="row.status | statusFilter">
            {{ row.status }}
          </el-tag>
        </template>
      </el-table-column>
      <el-table-column
        label="Acciones"
        align="center"
        width="230"
        class-name="small-padding fixed-width"
      >
        <template slot-scope="{ row, $index }">
          <el-button
            type="primary"
            size="mini"
            icon="el-icon-edit"
            @click="handleUpdate(row)"
          >
            Editar
          </el-button>
          <el-button
            v-if="row.status != 'eliminado'"
            size="mini"
            type="danger"
            icon="el-icon-delete"
            @click="handleDelete(row, $index)"
          >
            Borrar
          </el-button>
        </template>
      </el-table-column>
    </el-table>

    <pagination
      v-show="total > 0"
      :total="total"
      :page.sync="listQuery.page"
      :limit.sync="listQuery.limit"
      @pagination="getList"
    />

    <el-dialog :title="textMap[dialogStatus]" :visible.sync="dialogFormVisible">
      <el-row lg="12" sm="12">
        <el-form
          ref="dataForm"
          :rules="rules"
          :model="temp"
          label-position="left"
          label-width="120px"
          style="width: 500px"
        >
          <el-form-item label="Tipo de Dato" prop="type">
            <el-select v-model="temp.type" placeholder="Tipo de Dato">
              <el-option
                v-for="item in calendarTypeOptions"
                :key="item.key"
                :label="item.display_name"
                :value="item.key"
              />
            </el-select>
          </el-form-item>
          <el-form-item label="Fecha" prop="timestamp">
            <el-date-picker
              v-model="temp.timestamp"
              type="datetime"
              placeholder="Fecha"
            />
          </el-form-item>
          <el-form-item label="Descripcion" prop="title">
            <el-input v-model="temp.title" />
          </el-form-item>
          <el-form-item label="Estado">
            <el-select
              v-model="temp.status"
              class="filter-item"
              placeholder="Estado"
            >
              <el-option
                v-for="item in statusOptions"
                :key="item"
                :label="item"
                :value="item"
              />
            </el-select>
          </el-form-item>
          <el-form-item label="Comentarios">
            <el-input
              v-model="temp.remark"
              :autosize="{ minRows: 2, maxRows: 4 }"
              type="textarea"
              placeholder="Comentarios"
            />
          </el-form-item>
        </el-form>
      </el-row>

      <div slot="footer" class="dialog-footer">
        <el-button @click="dialogFormVisible = false">Cancelar</el-button>
        <el-button
          type="primary"
          @click="dialogStatus === 'create' ? createData() : updateData()"
          >OK</el-button
        >
      </div>
    </el-dialog>
  </div>
</template>

<script>
import {
  fetchList,
  fetchPv,
  createArticle,
  updateArticle,
} from "@/api/article";
import waves from "@/directive/waves"; // waves directive
import { parseTime } from "@/utils";
import Pagination from "@/components/Pagination"; // secondary package based on el-pagination

const calendarTypeOptions = [
  { key: "DT", display_name: "Fecha" },
  { key: "MO", display_name: "Moneda" },
  { key: "IN", display_name: "Numero Entero" },
  { key: "DE", display_name: "Numero Decimal" },
  { key: "OP", display_name: "Opciones" },
  { key: "PO", display_name: "Porcentaje" },
  { key: "SN", display_name: "Si/No" },
  { key: "TX", display_name: "Texto" },
];

// arr to obj, such as { CN : "China", US : "USA" }
const calendarTypeKeyValue = calendarTypeOptions.reduce((acc, cur) => {
  acc[cur.key] = cur.display_name;
  return acc;
}, {});

export default {
  name: "ComplexTable",
  components: { Pagination },
  directives: { waves },
  filters: {
    statusFilter(status) {
      const statusMap = {
        publicado: "success",
        borrador: "info",
        eliminado: "danger",
      };
      return statusMap[status];
    },
    typeFilter(type) {
      return calendarTypeKeyValue[type];
    },
  },
  data() {
    return {
      tableKey: 0,
      list: null,
      total: 0,
      listLoading: true,
      listQuery: {
        page: 1,
        limit: 20,
        importance: undefined,
        title: undefined,
        type: undefined,
        sort: "+id",
      },
      importanceOptions: [1, 2, 3],
      calendarTypeOptions,
      sortOptions: [
        { label: "ID ASC", key: "+id" },
        { label: "ID DESC", key: "-id" },
      ],
      statusOptions: ["publicado", "borrador", "eliminado"],
      showReviewer: false,
      temp: {
        id: undefined,
        importance: 1,
        remark: "",
        timestamp: new Date(),
        title: "",
        type: "",
        status: "publicado",
      },
      dialogFormVisible: false,
      dialogStatus: "",
      textMap: {
        update: "Editar",
        create: "Nuevo Registro",
      },
      dialogPvVisible: false,
      pvData: [],
      // Validacion de imput
      rules: {
        type: [
          {
            required: true,
            message: "Tipo de datos Requerido",
            trigger: "change",
          },
        ],
        timestamp: [
          {
            type: "date",
            required: true,
            message: "Fecha es requerido",
            trigger: "change",
          },
        ],
        title: [
          {
            required: true,
            message: "Descripcion es necesaria",
            trigger: "blur",
          },
        ],
      },
      downloadLoading: false,
    };
  },
  created() {
    this.getList();
  },
  methods: {
    getList() {
      this.listLoading = true;
      fetchList(this.listQuery).then((response) => {
        this.list = response.data.items;
        this.total = response.data.total;
        // Just to simulate the time of the request
        /* setTimeout(() => {
          this.listLoading = false
        }, 1.5 * 1000) */
        this.listLoading = false;
      });
    },
    handleFilter() {
      this.listQuery.page = 1;
      this.getList();
    },
    handleModifyStatus(row, status) {
      this.$message({
        message: "Exito Modificando",
        type: "success",
      });
      row.status = status;
    },
    sortChange(data) {
      const { prop, order } = data;
      if (prop === "id") {
        this.sortByID(order);
      }
    },
    sortByID(order) {
      if (order === "ascending") {
        this.listQuery.sort = "+id";
      } else {
        this.listQuery.sort = "-id";
      }
      this.handleFilter();
    },
    resetTemp() {
      this.temp = {
        id: undefined,
        importance: 1,
        remark: "",
        timestamp: new Date(),
        title: "",
        status: "publicado",
        type: "",
      };
    },
    handleCreate() {
      this.resetTemp();
      this.dialogStatus = "create";
      this.dialogFormVisible = true;
      this.$nextTick(() => {
        this.$refs["dataForm"].clearValidate();
      });
    },
    createData() {
      this.$refs["dataForm"].validate((valid) => {
        if (valid) {
          this.temp.id = parseInt(Math.random() * 100) + 1024; // mock a id
          this.temp.author = "vue-element-admin";
          createArticle(this.temp).then(() => {
            this.list.unshift(this.temp);
            this.dialogFormVisible = false;
            this.$notify({
              title: "Success",
              message: "Created Successfully",
              type: "success",
              duration: 1000,
            });
          });
        }
      });
    },
    handleUpdate(row) {
      this.temp = Object.assign({}, row); // copy obj
      this.temp.timestamp = new Date(this.temp.timestamp);
      this.dialogStatus = "update";
      this.dialogFormVisible = true;
      this.$nextTick(() => {
        this.$refs["dataForm"].clearValidate();
      });
    },
    updateData() {
      this.$refs["dataForm"].validate((valid) => {
        if (valid) {
          const tempData = Object.assign({}, this.temp);
          tempData.timestamp = +new Date(tempData.timestamp); // change Thu Nov 30 2017 16:41:05 GMT+0800 (CST) to 1512031311464
          updateArticle(tempData).then(() => {
            const index = this.list.findIndex((v) => v.id === this.temp.id);
            this.list.splice(index, 1, this.temp);
            this.dialogFormVisible = false;
            this.$notify({
              title: "Exito",
              message: "Al Guardar el registro",
              type: "success",
              duration: 1000,
            });
          });
        }
      });
    },
    handleDelete(row, index) {
      this.$notify({
        title: "Exito",
        message: "Registro Eliminado ",
        type: "success",
        duration: 1000,
      });
      this.list.splice(index, 1);
    },
    handleFetchPv(pv) {
      fetchPv(pv).then((response) => {
        this.pvData = response.data.pvData;
        this.dialogPvVisible = true;
      });
    },
    handleDownload() {
      this.downloadLoading = true;
      import("@/vendor/Export2Excel").then((excel) => {
        const tHeader = ["timestamp", "title", "type", "importance", "status"];
        const filterVal = [
          "timestamp",
          "title",
          "type",
          "importance",
          "status",
        ];
        const data = this.formatJson(filterVal);
        excel.export_json_to_excel({
          header: tHeader,
          data,
          filename: "table-list",
        });
        this.downloadLoading = false;
      });
    },
    formatJson(filterVal) {
      return this.list.map((v) =>
        filterVal.map((j) => {
          if (j === "timestamp") {
            return parseTime(v[j]);
          } else {
            return v[j];
          }
        })
      );
    },
    getSortClass: function (key) {
      const sort = this.listQuery.sort;
      return sort === `+${key}` ? "ascending" : "descending";
    },
  },
};
</script>
