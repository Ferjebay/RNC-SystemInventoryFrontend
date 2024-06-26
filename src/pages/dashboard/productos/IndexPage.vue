<script setup>
import { ref, onMounted, watch } from 'vue'
import ModalCargarExcel from "./components/ModalCargarExcel.vue";
import AddProduct from './AddProduct.vue'
import EditProduct from './EditProduct.vue'
import { useProduct } from "./composables/useProducts";
import useHelpers from "../../../composables/useHelpers";
import useRolPermisos from "src/composables/useRolPermisos.js";

  let {
    actualizarTabla,
    claim,
    modalAgregarProducto,
    modalEditarProducto,
    formProduct,
    selectSucursal: selectSucursalForm
  } = useProduct();

  const columns = [
    { name: 'acciones', label: 'acciones', align: 'center' },
    { name: 'codigoBarra', label: 'Codigo de Barra', align: 'center', field: 'codigoBarra' },
    { name: 'producto', label: 'Producto', align: 'center', field: 'nombre' },
    { name: 'stock', label: 'Stock', align: 'center', field: 'stock' },
    { name: 'descuento', label: 'Descuento', align: 'center', field: 'descuento' },
    { name: 'aplicaIva', label: 'Aplica Iva', align: 'center', field: 'aplicaIva' },
    { name: 'precio_compra', label: 'Precio Compra', align: 'center', field: 'precio_compra' },
    { name: 'pvp', label: 'PVP', align: 'center', field: 'pvp' },
    { name: 'estado', label: 'Estado', align: 'center', field: 'isActive' }
  ]

  const showModalUploadFile = ref( false );
  const { api, mostrarNotify, confirmDelete, isDeleted } = useHelpers();
  const { validarPermisos } = useRolPermisos();

  const rows           = ref([]);
  const tipoBusqueda   = ref('codigo');
  const filter         = ref('');
  const tableRef       = ref();
  const loading        = ref(false);
  const selectSucursal = ref('');
  const listSucursales = ref([]);

  const pagination = ref({
    sortBy: 'desc',
    descending: false,
    page: 1,
    rowsPerPage: 10,
    rowsNumber: 15
  })

  const activarDesactivarProduct = async (product_id, estado) => {
    try {
      const { data: { msg } } = await api.patch(`/products/${ product_id }/${ estado }`)
      mostrarNotify('positive', msg );
      getArticulos(1, pagination.value.rowsPerPage)
    } catch (error) {
      console.log(error);
    }
  }

  watch( isDeleted, ( newValue, _ ) => {
    if ( newValue ) getArticulos(1, pagination.value.rowsPerPage)
  })

  watch(tipoBusqueda, (currentValue, _) => {
    getArticulos(1, pagination.value.rowsPerPage);
  });

  watch(actualizarTabla, (currentValue, _) => {
    if ( currentValue ){
      getArticulos(1, pagination.value.rowsPerPage)
      actualizarTabla.value = false
    }
  });

  const eliminarProducto = async ( producto_id ) => {
    try {
      confirmDelete('Estas seguro de eliminar este producto?', `/products/${ producto_id }`);
    } catch (error) {
      console.log(error);
    }
  }

  const getSucursales = async () => {
    loading.value = true;
    try {
      const { data } = await api.get(`/sucursal/find/${ claim.company.id }/company`);
      data.forEach(( sucursal ) => {
        listSucursales.value.push({
          label:  sucursal.nombre,
          value:  sucursal.id
        })
      });
      if ( listSucursales.value.length !== 0 ) selectSucursal.value = listSucursales.value[0].value
    } catch ( error ) {
      mostrarNotify( 'warning', error.response.data.message )
    }
    loading.value = false;
  }

  const getArticulos = async (page = 1, rowsPerPage = 10) => {

    if ( listSucursales.value.length == 0 ) await getSucursales();

    let headers = { 'sucursal-id': selectSucursal.value };

    try {
      const { data } = await api.get('/products', {
        params: { page, limit: rowsPerPage, busqueda: filter.value },
        headers: headers
      })
      pagination.value.rowsNumber = data.meta.totalItems;
      rows.value = data.items;
    } catch (error) {
      console.log(error)
    }
  }

  const exportarProductos = async () => {
    try {

      const { data } = await api.post(`/products/download-products-excel`, {
        sucursal_id: selectSucursal.value
       }, {
        responseType: 'arraybuffer'
      });

      const blob = new Blob([ data ], {
        type: 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
      });
      const link = document.createElement('a');
      link.href = window.URL.createObjectURL(blob);
      link.download = 'productos_servicios.xlsx';
      document.body.appendChild(link);
      link.click();
      document.body.removeChild(link);

    } catch (error) {}
  }

  async function onRequest ( props ) {

    const { page, rowsPerPage, sortBy, descending } = props.pagination;

    loading.value = true

    await getArticulos( page, rowsPerPage );

    pagination.value.page        = page
    pagination.value.rowsPerPage = rowsPerPage
    pagination.value.sortBy      = sortBy
    pagination.value.descending  = descending

    loading.value = false
  }

  const downloadFile = () => {
    var archivoURL = "/plantillas/productos_plantilla.xlsx";

    window.location.href = archivoURL;
  }

  onMounted(() => {
    tableRef.value.requestServerInteraction()
  })

  const mode = ref("list");
</script>

<template>
  <div class="q-ma-lg q-pt-md">
    <div class="row q-col-gutter-lg">
      <div class="col-12">
        <q-card flat class="shadow_custom">
            <q-table
              title-class="text-grey-7 text-h6"
              :rows="rows"
              :loading="loading"
              :hide-header="mode === 'grid'"
              :columns="columns"
              row-key="name"
              :grid="mode==='grid'"
              :filter="filter"
              v-model:pagination="pagination"
              :rows-per-page-options="[10, 15, 20, 0]"
              ref="tableRef"
              binary-state-sort @request="onRequest">

              <template v-slot:loading>
                <q-inner-loading showing color="primary" />
              </template>

              <template v-slot:header="props">
                <q-tr :props="props" style="height: 60px">
                  <q-th v-for="col in props.cols"
                    :key="col.name" :props="props"
                    class="text-grey-7 text-weight-bold text-uppercase"
                    style="font-size: 13px">
                    {{ col.label }}
                  </q-th>
                </q-tr>
              </template>

              <template v-slot:top-left="props">
                <div v-if="claim.roles[0] !== 'SUPER-ADMINISTRADOR' && claim.roles[0] !== 'ADMINISTRADOR'"
                  class="text-center row justify-center" style="width: 100%;">
                  <label class="q-mb-sm text-grey-7 text-h6">
                    Listado de Productos
                  </label>
                </div>
                <div v-if="claim.roles[0] == 'SUPER-ADMINISTRADOR' || claim.roles[0] == 'ADMINISTRADOR'"
                style="display: flex" :class="[ $q.screen.xs ? 'q-mb-md' : '' ]">
                  <label class="q-mr-sm row items-center">
                    <span>Sucursal: </span>
                  </label>
                  <q-select outlined dense v-model="selectSucursal"
                    @update:model-value="getArticulos(1, pagination.rowsPerPage)"
                    emit-value map-options
                  :options="listSucursales">
                  </q-select>
                </div>
              </template>

              <template v-slot:top-right="props">
                <q-btn v-if="!$q.screen.xs && validarPermisos('crear.productos')"
                  @click="modalAgregarProducto = !modalAgregarProducto"
                  outline color="primary" label="Agregar Producto" class="q-mr-xs"/>

                <q-btn-dropdown outline class="q-mr-sm q-ml-xs"
                  color="teal-6" icon="fa-solid fa-file-excel">
                  <q-list>
                    <q-item clickable v-close-popup
                      @click="showModalUploadFile = true">
                      <q-item-section>
                        <q-item-label>Importar Excel</q-item-label>
                      </q-item-section>
                    </q-item>

                    <q-item @click="downloadFile"
                      clickable v-close-popup>
                      <q-item-section>
                        <q-item-label>Exportar Plantilla</q-item-label>
                      </q-item-section>
                    </q-item>

                    <q-item @click="exportarProductos"
                      clickable v-close-popup>
                      <q-item-section>
                        <q-item-label>Exportar Productos/Servicios</q-item-label>
                      </q-item-section>
                    </q-item>

                  </q-list>
                </q-btn-dropdown>

                <q-input :style="$q.screen.width > 700 || 'width: 70%'"
                  outlined dense debounce="900" v-model="filter" placeholder="Buscar...">
                  <template v-slot:append>
                    <q-icon name="search"/>
                  </template>
                </q-input>

                <q-btn flat round dense
                  :icon="props.inFullscreen ? 'fullscreen_exit' : 'fullscreen'"
                  @click="props.toggleFullscreen"
                  v-if="mode === 'list'" >
                  <q-tooltip :disable="$q.platform.is.mobile" v-close-popup>
                    {{ props.inFullscreen ? 'Exit Fullscreen' : 'Toggle Fullscreen' }}
                  </q-tooltip>
                </q-btn>

                <q-btn flat round dense
                  :icon="mode === 'grid' ? 'list' : 'grid_on'"
                  @click="mode = mode === 'grid' ? 'list' : 'grid'; separator = mode === 'grid' ? 'none' : 'horizontal'"
                  v-if="!props.inFullscreen"
                >
                  <q-tooltip :disable="$q.platform.is.mobile" v-close-popup>
                    {{ mode === 'grid' ? 'List' : 'Grid' }}
                  </q-tooltip>
                </q-btn>

              </template>

            <template v-slot:body-cell-acciones="props">
              <q-td :props="props">

                <template v-if="props.row.isActive">
                  <q-btn v-if="validarPermisos('editar.productos')"
                    round color="blue-grey"
                    @click="formProduct = { ...props.row,  }, selectSucursalForm = props.row.sucursal_id.id, modalEditarProducto = true"
                    icon="edit" class="q-mr-sm" size="10px" />
                  <q-btn round color="blue-grey"
                    v-if="validarPermisos('inactivar.productos')"
                    icon="close"
                    @click="activarDesactivarProduct(props.row.id, false)"
                    size="10px" />
                </template>

                <template v-else>
                  <q-btn round color="blue-grey"
                    v-if="validarPermisos('activar.productos')"
                    icon="done" size="10px"
                    @click="activarDesactivarProduct(props.row.id, true)" />

                  <q-btn round color="blue-grey" class="q-ml-sm"
                    v-if="!props.row.isActive && validarPermisos('eliminar.productos')"
                    icon="delete" size="10px"
                    @click="eliminarProducto(props.row.id)" />
                </template>
              </q-td>
            </template>
            <template v-slot:body-cell-codigoBarra="props">
              <q-td :props="props"> {{ props.row.codigoBarra }} </q-td>
            </template>
            <template v-slot:body-cell-producto="props">
              <q-td :props="props"> {{ props.row.nombre }} </q-td>
            </template>
            <template v-slot:body-cell-descuento="props">
              <q-td :props="props"> {{ props.row.descuento }}% </q-td>
            </template>
            <template v-slot:body-cell-aplicaIva="props">
              <q-td :props="props"> {{ props.row.aplicaIva ? 'SI' : 'NO' }} </q-td>
            </template>
            <template v-slot:body-cell-estado="props">
              <q-td :props="props">
                <template v-if="props.row.isActive">
                    <q-badge outline color="positive" label="Activo" class="q-pa-sm" />
                </template>
                <template v-else>
                    <q-badge outline color="red" label="Inactivo" class="q-pa-sm" />
                </template>
              </q-td>
            </template>

            <template v-slot:no-data="{ icon }">
              <div class="full-width row flex-center text-lime-10 q-gutter-sm">
                <span class="text-subtitle1">
                  No se encontró ningun Resultado
                </span>
              </div>
            </template>
          </q-table>
        </q-card>
      </div>
    </div>
  </div>

  <q-dialog v-model="modalAgregarProducto">
    <AddProduct />
  </q-dialog>

  <q-dialog v-model="modalEditarProducto">
    <EditProduct />
  </q-dialog>

  <q-page-sticky position="bottom-right" :offset="[18, 18]"
      v-if="$q.screen.xs && validarPermisos('crear.productos')">
    <q-btn round color="secondary" size="lg"
        icon="add" @click="modalAgregarProducto = true" />
  </q-page-sticky>

  <q-dialog v-model="showModalUploadFile">
    <ModalCargarExcel @actualizarDatos="getArticulos()" />
  </q-dialog>
</template>

