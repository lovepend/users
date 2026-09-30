<template>
  <DefaultCard :title="{ icon: 'fa-list', title: safeTranslate('menu.register.register_list_title') }">
    <template #actions v-if="focusDataRef && !isManagerDataRef"><NoPermissionArea /></template>
    <template #header-right v-else>
      <NewBasicToolBar v-bind="toolbarBindRef" v-on="toolbarEventsRef" />
    </template>
    <ag-grid-vue v-bind="gridBindRef" v-on="gridEvents" />
  </DefaultCard>
</template>

<script setup>
  // Library - Common
  import { ref, shallowRef, computed, onMounted, watch } from 'vue'
  import { useRoute } from 'vue-router'
  import { storeToRefs } from 'pinia'

  // Library - Utils
  import * as _ from 'lodash-es'
  import BigNumber from 'bignumber.js'
  import { v4 as uuidv4 } from 'uuid'

  // Store
  import { useAppAuthenticationStore } from '@/stores/app-authentication'
  import { useAppPermissionStore } from '@/stores/app-permission'
  import { useAppMenuStore } from '@/stores/app-menu'
  import { useEmployees } from '@/stores/app-employees'

  // Helpers
  import { ApiCommon } from '@/utils/axios/api-common'
  import { dataUtil as DU } from '@/utils/functions/data-utility'
  import { toast } from '@/composables/useToast'
  import useI18nExtends from '@/composables/useI18nExtends'
  import global from '@/constants/enums/global'

  // Custom Component
  import NoPermissionArea from '@/components/editors/button/NoPermissionArea.vue'
  import NewBasicToolBar from '@/components/toolBar/NewBasicToolBar.vue'
  import AggridLoadingOverlay from '@/components/aggrid/AgGridLoadingOverlay.vue'
  import AgDetailStatusBar from '@/components/aggrid/AgDetailStatusBar.vue'
  import AgCustomFilter from '@/components/aggrid/AgCustomFilter.vue'
  import AgComboEditor from '@/components/aggrid/AgComboEditor.vue'

  // GLOBAL
  const route = useRoute()
  const { safeTranslate } = useI18nExtends()
  const { userInfo } = storeToRefs(useAppAuthenticationStore())
  const { isSetupAdmin, projectPermissions } = storeToRefs(useAppPermissionStore())
  const { currentProjNo, currentEpId } = storeToRefs(useAppMenuStore())
  const { getEmployeeById, getNameDeptKey } = useEmployees()

  const _Emits = defineEmits([
    'show-overlay',
    'hide-overlay',
    'on-pre-initialize',
    'on-sub-refresh',
    'on-sub-stop-editing',
    // Individual Execution
    'on-update-edit-pub',
    'on-map-cls-refresh',
    'on-lnk-att-refresh',
    'on-map-att-refresh',
    'on-join-table-refresh',
    'on-map-cls-refresh-cells',
    'on-lnk-att-refresh-cells',
    'on-map-att-refresh-cells',
    'on-join-table-refresh-cells',
    'on-map-cls-stop-editing',
    'on-lnk-att-stop-editing',
    'on-map-att-stop-editing',
    'on-join-table-stop-editing',
    // 'on-map-rel-refresh',
    // 'on-map-rel-refresh-cells',
    // 'on-map-rel-stop-editing',
  ])
  const _Props = defineProps({
    // UTILS
    REG_UTILS: { type: Object, required: false, default: null },
    JOIN_UTILS: { type: Object, required: false, default: null },
    IS_CONVERT_BY_UOM_FACTOR_LNK: { type: Boolean, required: false, default: false },
    IS_CONVERT_BY_UOM_FACTOR_MAP: { type: Boolean, required: false, default: false },
    // PRE_LOAD_DATA
    PROJ_LIST: { type: [Array, Object], required: false, default: () => [] },
    PROJ_DICT: { type: Object, required: false, default: () => ({}) },
    EP_LIST: { type: [Array, Object], required: false, default: () => [] },
    EP_DICT: { type: Object, required: false, default: () => ({}) },
    CLS_LIST: { type: [Array, Object], required: false, default: () => [] },
    CLS_DICT: { type: Object, required: false, default: () => ({}) },
    TAG_TYPE_LIST: { type: [Array, Object], required: false, default: () => [] },
    TAG_TYPE_DICT: { type: Object, required: false, default: () => ({}) },
    UOM_LIST: { type: [Array, Object], required: false, default: () => [] },
    UOM_TREE: { type: [Array, Object], required: false, default: () => [] },
    UOM_DICT: { type: Object, required: false, default: () => ({}) },
    ATT_DICT: { type: Object, required: false, default: () => ({}) },
    DEF_ATT_LIST: { type: [Array, Object], required: false, default: () => [] },
    DEF_ATT_DICT: { type: Object, required: false, default: () => ({}) },
    PBS_SYS_LIST: { type: [Array, Object], required: false, default: () => [] },
    PBS_SYS_DICT: { type: Object, required: false, default: () => ({}) },
    PBS_ASM_LIST: { type: [Array, Object], required: false, default: () => [] },
    PBS_ASM_DICT: { type: Object, required: false, default: () => ({}) },
    // SUB_DATA
    IS_SUB_EDITED: { type: [Boolean, Object], required: false, default: false },
    // SUB_DATA_INDIVIDUAL
    IS_MAP_CLS_EDITED: { type: [Boolean, Object], required: false, default: false },
    IS_LNK_ATT_EDITED: { type: [Boolean, Object], required: false, default: false },
    IS_MAP_ATT_EDITED: { type: [Boolean, Object], required: false, default: false },
    IS_JOIN_TABLE_EDITED: { type: [Boolean, Object], required: false, default: false },
    MAP_CLS_DATA_LIST: { type: [Array, Object], required: false, default: () => [] },
    LNK_ATT_DATA_LIST: { type: [Array, Object], required: false, default: () => [] },
    MAP_ATT_DATA_LIST: { type: [Array, Object], required: false, default: () => [] },
    JOIN_TABLE_DATA_LIST: { type: [Array, Object], required: false, default: () => [] },
    IS_MAP_REL_EDITED: { type: [Boolean, Object], required: false, default: false },
    MAP_REL_DATA_LIST: { type: [Array, Object], required: false, default: () => [] },
  })

  // UTILS
  const regUtilsRef = computed(() => _Props.REG_UTILS)
  const joinUtilsRef = computed(() => _Props.JOIN_UTILS)
  const isConvertByUomFactorLnkRef = computed(() => _Props.IS_CONVERT_BY_UOM_FACTOR_LNK) // DEF_VAL, MIN_VAL, MAX_VAL
  const isConvertByUomFactorMapRef = computed(() => _Props.IS_CONVERT_BY_UOM_FACTOR_MAP) // OPER-VALUE
  // PRE_LOAD_DATA
  const projListRef = computed(() => _Props.PROJ_LIST)
  const projDictRef = computed(() => _Props.PROJ_DICT)
  const epListRef = computed(() => _Props.EP_LIST)
  const epDictRef = computed(() => _Props.EP_DICT)
  const clsListRef = computed(() => _Props.CLS_LIST)
  const clsDictRef = computed(() => _Props.CLS_DICT)
  const tagTypeListRef = computed(() => _Props.TAG_TYPE_LIST)
  const tagTypeDictRef = computed(() => _Props.TAG_TYPE_DICT)
  const uomListRef = computed(() => _Props.UOM_LIST)
  const uomTreeRef = computed(() => _Props.UOM_TREE)
  const uomDictRef = computed(() => _Props.UOM_DICT)
  const attDictRef = computed(() => _Props.ATT_DICT)
  const defAttListRef = computed(() => _Props.DEF_ATT_LIST)
  const defAttDictRef = computed(() => _Props.DEF_ATT_DICT)
  const pbsSysListRef = computed(() => _Props.PBS_SYS_LIST)
  const pbsSysDictRef = computed(() => _Props.PBS_SYS_DICT)
  const pbsAsmListRef = computed(() => _Props.PBS_ASM_LIST)
  const pbsAsmDictRef = computed(() => _Props.PBS_ASM_DICT)
  // SUB_DATA
  const isSubEditedRef = computed(() => _Props.IS_SUB_EDITED)
  // SUB_DATA_INDIVIDUAL
  const isMapClsEditedRef = computed(() => _Props.IS_MAP_CLS_EDITED)
  const isLnkAttEditedRef = computed(() => _Props.IS_LNK_ATT_EDITED)
  const isMapAttEditedRef = computed(() => _Props.IS_MAP_ATT_EDITED)
  const isJoinTableEditedRef = computed(() => _Props.IS_JOIN_TABLE_EDITED)
  const mapClsDataListRef = computed(() => _Props.MAP_CLS_DATA_LIST)
  const lnkAttDataListRef = computed(() => _Props.LNK_ATT_DATA_LIST)
  const mapAttDataListRef = computed(() => _Props.MAP_ATT_DATA_LIST)
  const joinTableDataListRef = computed(() => _Props.JOIN_TABLE_DATA_LIST)
  const isMapRelEditedRef = computed(() => _Props.IS_MAP_REL_EDITED)
  const mapRelDataListRef = computed(() => _Props.MAP_REL_DATA_LIST)
  watch(isSubEditedRef, item => {
    if (item) {
      const grid = gridRef.value
      const focusData = focusDataRef.value
      const focusDataId = focusData?._ROW_ID
      grid?.getRowNode(focusDataId)?.setSelected(true)
      if (focusData) {
        focusData._STATUS = focusData._STATUS || 'M'
        onRefreshCells({ columns: ['_STATUS'] })
      }
    }
  })

  const firstFocusIDRef = computed(() => route.query?.reg_type_id)
  const onShowOverlay = config => _Emits('show-overlay', config)
  const onHideOverlay = () => _Emits('hide-overlay')
  const onShowLoading = () => gridRef.value?.setGridOption('loading', true)
  const onHideLoading = () => gridRef.value?.setGridOption('loading', false)

  // TOOLBAR BUTTON
  const toolbarBindRef = computed(() => ({
    // ---------- Button Wrapper Style ----------------------------------------
    toolbarClass: {},
    toolbarStyle: {},
    // areaGap: 10,
    // ---------- BUTTON - Visible --------------------------------------------
    showCreate: true,
    showMultiCreate: true,
    showDelete: true,
    // showUpdate: true,
    showCopy: true,
    // showRefresh: true,
    // showSave: true,
    // ---------- BUTTON - Disabled -------------------------------------------
    // blockCreate: false,
    // blockMultiCreate: false,
    // blockDelete: false,
    // blockUpdate: false,
    // blockCopy: false,
    // blockRefresh: false,
    // blockSave: false,
  }))
  const toolbarEventsRef = computed(() => ({
    execCreate: onCreateRows,
    execMultiCreate: onCreateMultiRows,
    execDelete: onDeleteRows,
    // execUpdate: onUpdate,
    execCopy: onCopyRegister,
    // execSave: saveData,
    // execRefresh: onInitialize,
  }))

  // MODIFIED DATA
  // KEY ID Update용 { [_ROW_ID]: { _ROW_ID: [_ROW_ID], KEY: '' } } // KEY - Original Key
  const modKeysRef = shallowRef({})
  const modRowsRef = shallowRef(null)
  const IS_APPLY_MOD_SUB_DATA = false

  // FOCUS DATA
  const prevIndexRef = shallowRef(-1)
  const focusDataRef = ref()
  const focusDataOrgRef = computed(() => {
    const focusData = focusDataRef.value
    const rowId = focusData?._ROW_ID
    if (rowId) {
      const dataDict = gridDataOrgDictRef.value
      return dataDict?.[rowId]
    }
    return undefined
  })
  const isManagerDataRef = computed(() => {
    const focusData = focusDataRef.value
    if (focusData) {
      if (focusData._STATUS === 'A') {
        return true
      } else if (focusData.EP_ID && focusData.PROJ_NO && focusData.PROJ_NO === currentProjNo.value) {
        if (isSetupAdmin.value) return true
        if (projectPermissions.value?.some(v => v.EP_ID === focusData.EP_ID && v.Role === 'Manager')) return true
      }
    }
    return false
  })

  // GRID
  const gridRef = shallowRef()
  const gridDataRef = shallowRef([])
  const gridDelDataRef = shallowRef([]) // DELETED == true의 데이터
  const gridCmplxDataRef = shallowRef([]) // 전체 TYPE_ID 확인용
  // 데이터 원본 확인용
  const gridDataOrgRef = shallowRef([])
  const gridDataOrgDictRef = shallowRef({}) // 데이터 원본 확인용 { _ROW_ID: data, ... }
  const gridDataRenderedRef = shallowRef(false)
  const gridInitializingRef = shallowRef(false)
  const gridFrameIdRef = shallowRef(0) // requestAnimationFrame내 Grid RefreshCells용 token
  const gridRefreshCellsOptionRef = shallowRef(null) // requestAnimationFrame내 Grid RefreshCells용 Option
  const dateFilterParams = {
    minValidDate: '2008-01-08',
    maxValidDate: new Date(new Date().getTime() + 24 * 60 * 60 * 1000),
    comparator: (filterLocalDateAtMidnight, cellValue) => {
      const dateStr = cellValue?.substring(0, 10)
      if (!dateStr) {
        return -1
      }
      const splitted = dateStr.split('-')
      const date = new Date(Number(splitted[0]), Number(splitted[1]) - 1, Number(splitted[2]))

      if (filterLocalDateAtMidnight.getTime() === date.getTime()) {
        return 0
      } else if (filterLocalDateAtMidnight > date) {
        return -1
      } else if (filterLocalDateAtMidnight < date) {
        return 1
      }
      return 0
    },
  }
  // prettier-ignore
  const gridColDefsRef = ref([
    {
      suppressHeaderMenuButton: true,
      // headerCheckboxSelectionFilteredOnly: true,
      minWidth: 30,
      maxWidth: 30,
      pinned: 'left',
      lockPinned: true,
      lockVisible: true,
      lockPosition: 'left',
      suppressMovable: true,
      resizable: false,
      sortable: false,
      editable: false,
      filter: false, // 'agSetColumnFilter',
      filterValueGetter: params => params.node.selected,
      filterParams: {
        suppressSorting: true,
        refreshValuesOnOpen: true,
      },
      checkboxSelection: params => {
        return (
          !params.node.group &&
          params.data?.PROJ_NO === currentProjNo.value && (
            !epDictRef.value?.[params.data?.EP_ID] ||
            isSetupAdmin.value ||
            projectPermissions.value?.some(v => v.EP_ID === params.data?.EP_ID && v.Role === 'Manager')
          )
        )
      },
      headerCheckboxSelection: false,
    },
    {
      field: '_STATUS',
      headerName: 'Status',
      hide: true,
      minWidth: 60,
      maxWidth: 60,
      pinned: 'left',
      lockPinned: true,
      // lockVisible: true,
      lockPosition: 'left',
      suppressMovable: true,
      resizable: false,
      cellClass: 'agReadOnly',
      editable: false,
      cellRenderer: params => {
        const value = params.value
        if (value) {
          const wrap = document.createElement('div')
          const badge = document.createElement('span')
          wrap.className = 'w-100 h-100 d-flex justify-content-center align-items-center'
          switch (value) {
            case 'A': {
              badge.className = `badge rounded-pill fs-12px bg-green-400 text-border-gray`
              badge.textContent = 'New'
            } break;
            case 'M': {
              badge.className = `badge rounded-pill fs-12px bg-orange-400 text-border-gray`
              badge.textContent = 'Mod' // Modified
            } break;
            case 'D': {
              badge.className = `badge rounded-pill fs-12px bg-red-400 text-border-gray`
              badge.textContent = 'Del' // Deleted
            } break;
          }
          wrap.append(badge)
          return wrap
        }
      },
    },
    {
      field: 'TYPE_ID',
      headerName: 'Register ID',
      pinned: 'left',
      lockPinned: true,
      lockVisible: true,
      lockPosition: 'left',
      minWidth: 120,
      flex: 5,
      // tooltipValueGetter: params => params.valueFormatted || params.value,
      valueSetter: params => {
        const dataItem = params.data
        if (dataItem) {
          dataItem.TYPE_ID = String(params.newValue ?? '').toUpperCase()
        }
        return true
      },
      valueFormatter: params => {
        return String(params.value ?? '').toUpperCase()
      },
      // cellRenderer: params => {
      //   const dataItem = params.data
      //   const value = String(params.value ?? '').toUpperCase()
      //   const textColor = (
      //     dataItem?.PROJ_NO !== currentProjNo.value ? 'text-orange-600' : ''
      //   )
      //   const iconColor = (
      //     dataItem?.PROJ_NO !== currentProjNo.value ? 'text-orange-600' :
      //     dataItem?._STATUS !== 'A' ? 'text-blue-400' : 'text-blue-800'
      //   )
      //   const box = document.createElement('div');
      //   const icon = document.createElement('i');
      //   const text = document.createElement('span');
      //   icon.className = `fa fa-tag me-1 ${iconColor}`;
      //   text.className = `${textColor}`;
      //   text.innerText = value;
      //   box.append(icon, text);
      //   return box;
      // },
    },
    {
      field: 'EP_ID',
      headerName: 'EP',
      minWidth: 60,
      flex: 1,
      cellStyle: params => {
        const value = params.value;
        if (value) {
          const epInfo = epDictRef.value?.[value]
          if (!epInfo || epInfo?.DELETED) {
            return {
              color: '#ff0000',
              textDecoration: 'line-through',
            }
          }
        }
        return { // 초기화 없으면 스타일 유지됨
          color: '',
          textDecoration: '',
        }
      },
      cellRenderer: params => {
        const displayValue = params.valueFormatted || params.value || ''
        const editableDef  = params.colDef?.editable;
        const editable     = (
          DU.isFunction(editableDef) ? editableDef(params) : editableDef
        );
        return makeComboRender({
          value   : displayValue,
          editable: editable,
        });
      },
      singleClickEdit: false,
      cellEditorPopup: true,
      cellEditorPopupPosition: 'over',
      cellEditor: 'AgComboEditor', // 'agRichSelectCellEditor'
      cellEditorParams: (params) => {
        let values = projectPermissions.value ?? []
        if (!isSetupAdmin.value) {
          values = values.filter(v => v.Role === 'Manager');
        }
        values = values.map(v => ({
          code : v.EP_ID,
          label: epDictRef.value?.[v.EP_ID]?.label || v.EP_ID,
        }))
        return {
          values            : values,
          searchType        : 'matchAny',
          highlightMatch    : true,
          valueListMaxHeight: 200,
          valueListMaxWidth : 200,
          // AgComboEditor Parameters
          placeholder         : 'Select EP', // Default = ''
          code                : 'code',  // Default = 'code'
          label               : 'label', // Default = 'label'
          pushTags            : false,   // Default = false
          multiple            : false,   // Default = false
          multipleCheck       : false,   // Default = false
          treeSelect          : false,   // Default = false
          closeOnSelect       : false,   // Default = true
          deselectFromDropdown: false,   // Default = false
          returnOrigin        : false,   // Default = false
          applySelected       : true,    // Default = false
          // getOptionLabel      : (v) => (`${v.label} (${v.code})`),
        }
      },
    },
    {
      field: 'DESC',
      headerName: 'Description',
      minWidth: 150,
      flex: 4,
      // tooltipValueGetter: params => params.valueFormatted || params.value,
    },
    {
      field: 'NEW_TAG_YN',
      headerName: 'Object Create',
      cellDataType: 'boolean', // Use Using valueParser
      width: 60,
      // tooltipValueGetter: params => params.valueFormatted || params.value,
      filter: 'agSetColumnFilter',
      filterValueGetter: params => params.data?.[params.colDef.field],
      cellStyle: { justifyItems: 'center' },
      valueParser: params => Boolean(params.newValue), // Use editable
      // cellEditor: 'agCheckboxCellEditor',
      editable: params => {
        const grid = params.api
        const dataItem = params.data
        const editableDef = grid.getGridOption('defaultColDef')?.editable
        const editable = DU.isFunction(editableDef) ? editableDef(params) : editableDef
        if (editable) return dataItem?.USE_WRK_YN
        return false
      },
    },
    {
      field: 'USE_WRK_YN',
      headerName: 'Working',
      cellDataType: 'boolean', // Use Using valueParser
      width: 60,
      // tooltipValueGetter: params => params.valueFormatted || params.value,
      filter: 'agSetColumnFilter',
      filterValueGetter: params => params.data?.[params.colDef.field],
      cellStyle: { justifyItems: 'center' },
      valueParser: params => Boolean(params.newValue), // Use editable
      // cellEditor: 'agCheckboxCellEditor',
    },
    // 2602012 개발팀 요청으로 All Tag 사용하지 않아서 우선 숨김 처리.
    // {
    //   field: 'ALL_TAG_YN',
    //   headerName: 'All Tag',
    //   hide: true,
    //   width: 60,
    //   // tooltipValueGetter: params => params.valueFormatted || params.value,
    // },
    {
      field: 'REMARK',
      headerName: 'Remark',
      minWidth: 150,
      flex: 3,
      // tooltipValueGetter: params => params.valueFormatted || params.value,
    },
    {
      field: 'SEQ',
      headerName: 'SEQ',
      cellDataType: 'number',
      hide: true,
      minWidth: 55,
      maxWidth: 55,
      resizable: false,
      // tooltipValueGetter: params => params.valueFormatted || params.value,
      valueParser: params => isNaN(params.newValue) ? 0 : Number(params.newValue),
      // cellEditor: 'agNumberCellEditor',
      cellEditorParams: {
        min: 0,
        max: 10000,
        step: 1,
      },
    },
    {
      field: 'PROJ_NO',
      headerName: 'Project',
      hide: true,
      // tooltipValueGetter: params => params.valueFormatted || params.value,
      cellClass: 'agReadOnly',
      editable: false,
    },
    {
      field: 'CRTER_NO',
      headerName: 'CRTER NO',
      hide: true,
      // tooltipValueGetter: params => params.valueFormatted || params.value,
      filterValueGetter: params => {
        const dataItem = params.data;
        const field = params.colDef.field
        const {
          NAME: nameKey,
          DEPT: deptKey
        } = getNameDeptKey(field) ?? {}
        if (nameKey && deptKey) {
          const EMP_NM = dataItem[nameKey]
          const DEPT_NAME = dataItem[deptKey]
          if (EMP_NM !== undefined) {
            return `${EMP_NM}${DEPT_NAME ? `(${DEPT_NAME})` : ''}`
          }
        }
        return dataItem[field]
      },
      cellClass: 'agReadOnly',
      editable: false,
      cellRenderer: params => {
        const dataItem = params.data;
        const value = params.value
        const field = params.colDef.field
        if (dataItem && value) {
          const {
            NAME: nameKey,
            DEPT: deptKey
          } = getNameDeptKey(field) ?? {}
          if (nameKey && deptKey) {
            const oldName = dataItem[nameKey] ?? ''
            const oldDept = dataItem[deptKey] ?? ''
            let newName = ''
            let newDept = ''
            dataItem[nameKey] = newName
            dataItem[deptKey] = newDept

            const span = document.createElement('span')
            getEmployeeById(value).then(empInfo => {
              newName = empInfo?.EMP_NM ?? ''
              newDept = empInfo?.DEPT_NAME ?? ''
              dataItem[nameKey] = newName
              dataItem[deptKey] = newDept
              span.textContent = `${newName}${newDept ? `(${newDept})` : ''}`
            }).finally(() => {
              if (oldName !== newName || oldDept !== newDept) {
                params.api.applyTransaction({ update: [dataItem] })
              }
            })
            return span
          }
        }
        return value
      }
    },
    {
      field: 'CRTE_DTM',
      headerName: 'CRTE DTM',
      cellDataType: 'date',
      hide: true,
      tooltipValueGetter: params => params.valueFormatted || params.value,
      filter: 'agDateColumnFilter',
      filterParams: dateFilterParams,
      cellClass: 'agReadOnly',
      editable: false,
      valueFormatter: (params) => { return dateFullFormatter(params) },
    },
    {
      field: 'CHGER_NO',
      headerName: 'CHGER NO',
      hide: true,
      // tooltipValueGetter: params => params.valueFormatted || params.value,
      filterValueGetter: params => {
        const dataItem = params.data;
        const field = params.colDef.field
        const {
          NAME: nameKey,
          DEPT: deptKey
        } = getNameDeptKey(field) ?? {}
        if (nameKey && deptKey) {
          const EMP_NM = dataItem[nameKey]
          const DEPT_NAME = dataItem[deptKey]
          if (EMP_NM !== undefined) {
            return `${EMP_NM}${DEPT_NAME ? `(${DEPT_NAME})` : ''}`
          }
        }
        return dataItem[field]
      },
      cellClass: 'agReadOnly',
      editable: false,
      cellRenderer: params => {
        const dataItem = params.data;
        const value = params.value
        const field = params.colDef.field
        if (dataItem && value) {
          const {
            NAME: nameKey,
            DEPT: deptKey
          } = getNameDeptKey(field) ?? {}
          if (nameKey && deptKey) {
            const oldName = dataItem[nameKey] ?? ''
            const oldDept = dataItem[deptKey] ?? ''
            let newName = ''
            let newDept = ''
            dataItem[nameKey] = newName
            dataItem[deptKey] = newDept

            const span = document.createElement('span')
            getEmployeeById(value).then(empInfo => {
              newName = empInfo?.EMP_NM ?? ''
              newDept = empInfo?.DEPT_NAME ?? ''
              dataItem[nameKey] = newName
              dataItem[deptKey] = newDept
              span.textContent = `${newName}${newDept ? `(${newDept})` : ''}`
            }).finally(() => {
              if (oldName !== newName || oldDept !== newDept) {
                params.api.applyTransaction({ update: [dataItem] })
              }
            })
            return span
          }
        }
        return value
      }
    },
    {
      field: 'CHGE_DTM',
      headerName: 'CHGE DTM',
      cellDataType: 'date',
      hide: true,
      tooltipValueGetter: params => params.valueFormatted || params.value,
      filter: 'agDateColumnFilter',
      filterParams: dateFilterParams,
      cellClass: 'agReadOnly',
      editable: false,
      valueFormatter: (params) => { return dateFullFormatter(params) },
    },
    { field: '_id', headerName: 'id', hide: true, editable: false, cellClass: 'agReadOnly' },
  ])

  // prettier-ignore
  const onInitialize = async function () {
    const onFinally = function () {
      // Execute after data-binding
      setTimeout(() => {
        if (gridDataRenderedRef.value) { setFocusData(true); }
        onRefreshCells()
        onHideLoading()
        onHideOverlay()
        gridDataRenderedRef.value = true
        setTimeout(() => (gridInitializingRef.value = false), 0)
      }, 10)
    }
    gridInitializingRef.value = true

    onShowOverlay()
    onShowLoading()
    // onResetSelection()
    prevIndexRef.value = -1
    focusDataRef.value = null
    modRowsRef.value = null
    modKeysRef.value = {}
    gridDataRef.value = []
    gridDelDataRef.value = []
    gridDataOrgRef.value = []
    gridDataOrgDictRef.value = {}
    gridCmplxDataRef.value = []
    if (!currentProjNo.value) { return onFinally(); }

    let response
    try {
      response = await ApiCommon.postRegisterGet({ ProjectNo: currentProjNo.value })
    } catch (error) {
      console.error(error);
      toast.fail({
        okText: '확인',
        title: '오류',
        content: String(error.response?.data?.Message || error.message).replace(/\r\n/g, '<br>'),
      })
      return onFinally()
    }

    const dataList = []
    const delDataList = []
    const orgDataList = []
    const orgDataDict = {}
    const cmplxList = []
    response?.forEach((data, index) => {
      if (!data.CMPLX_YN) {
        data._ROW_ID = `ROW_${String(index).padStart(6, '0')}`
        const orgData = _.cloneDeep(data)
        orgDataDict[orgData._ROW_ID] = orgData
        orgDataList.push(orgData)
        if (!data.DELETED) {
          dataList.push(data)
        } else {
          data._STATUS = 'D'
          delDataList.push(data)
        }
      } else {
        cmplxList.push(data)
      }
    })
    gridDataRef.value = dataList
    gridDelDataRef.value = delDataList
    gridDataOrgRef.value = orgDataList
    gridDataOrgDictRef.value = orgDataDict
    gridCmplxDataRef.value = cmplxList

    return onFinally()
  }
  const onInitializeDebounce = _.debounce(onInitialize, 100)

  const onCreateRows = function () {
    if (!currentProjNo.value) {
      toast.warning({ title: `Project를  선택해주세요`, duration: 5000 })
    } else {
      insertRows(1)
    }
  }

  const onCreateMultiRows = function (rowCount) {
    if (!currentProjNo.value) {
      toast.warning({ title: `Project를  선택해주세요`, duration: 5000 })
    } else {
      insertRows(rowCount)
    }
  }

  const insertRows = function (rowCount) {
    const grid = gridRef.value
    grid.stopEditing()
    onShowLoading()

    const addRows = []
    for (let i = 0; i < rowCount; i++) {
      const newId = `NEW_ROW_${Math.random().toString(36).substring(2, 12)}`
      addRows.push({
        _ROW_ID: newId,
        _STATUS: 'A',

        TYPE_ID: null,
        DESC: '',
        SEQ: 0,
        EP_ID: null,
        ALL_TAG_YN: false,
        NEW_TAG_YN: true,
        USE_WRK_YN: true,
        CMPLX_YN: false,
        CMPL_SETT: null,
        MAP_CLS_ID: [],
        JOIN_TABLS: [],
        LNK_ATT: [],
        MAP_ATT: [],
        MAP_REL: [],
        PROJ_NO: currentProjNo.value,
      })
    }
    if (addRows.length) {
      gridDataRef.value.push(...addRows)
      // 생성시 Sub항목 수정 가능
      addRows.forEach(v => {
        const item = { ...v }
        gridDataOrgRef.value.push(item)
        gridDataOrgDictRef.value[v._ROW_ID] = item
      })
      const lastIndex = grid.getDisplayedRowCount()
      const result = grid.applyTransaction({ add: addRows, addIndex: lastIndex })
      onRefreshCells()
      grid.clearRangeSelection() // ~32.0 - deprecated
      grid.clearFocusedCell()
      grid.setFocusedCell(lastIndex, 'TYPE_ID')
      // grid.setNodesSelected({
      //   nodes: result.add,
      //   newValue: true,
      // })
      grid.ensureIndexVisible(lastIndex, 'bottom')
    }
    onHideLoading()
  }

  const onDeleteRows = async function () {
    const grid = gridRef.value
    const selectedRows = grid?.getSelectedRows()
    if (!selectedRows || !selectedRows.length) {
      toast.warning({ title: `Please select the data you want to delete.`, duration: 5000 })
      return
    }

    if (!isSetupAdmin.value) {
      for (let i = 0; i < selectedRows.length; i++) {
        const selectRow = selectedRows[i]
        const rowId = selectRow._ROW_ID
        const orgData = gridDataOrgDictRef.value?.[rowId]
        const epId = orgData?.EP_ID || selectRow.EP_ID
        if (!projectPermissions.value?.some(v => v.EP_ID === epId && v.Role === 'Manager')) {
          toast.warning({ title: `권한이 없는 EP[${epId}] 항목은 삭제할 수 없습니다.`, duration: 5000 })
          isBreak = true
          return
        }
      }
    }

    toast.warning({
      title: 'It will be deleted immediately. Are you sure you want to delete?',
      okText: 'OK',
      cancelText: 'Cancel',
      onOk: () => onDeleteRowsConfirmed(grid, selectedRows),
    })
  }

  const onDeleteRowsConfirmed = async function (grid, selectedRows) {
    const onFinally = function () {
      onHideOverlay()
    }
    onShowOverlay()
    await new Promise(r => setTimeout(r, 10))

    const passList = []
    const failList = []
    const promises_all = []
    const promises_race = []
    for (let i = 0; i < selectedRows.length; i++) {
      const dataItem = selectedRows[i]
      const { TYPE_ID, _STATUS, _ROW_ID: rowId } = dataItem
      let delKeyId = dataItem?.TYPE_ID

      if (promises_race.length >= 20) {
        await Promise.race(promises_race)
      }
      if (_STATUS !== 'A') {
        let orgKeyInfo = modKeysRef.value[rowId]
        if (orgKeyInfo) {
          if (orgKeyInfo.KEY !== TYPE_ID) {
            delKeyId = orgKeyInfo.KEY
          } else {
            delete modKeysRef.value[rowId]
            orgKeyInfo = null
          }
        }

        const promise = ApiCommon.postRegisterDelete({
          ProjectNo: currentProjNo.value,
          TYPE_ID: delKeyId,
        })
          .then(() => {
            passList.push(dataItem)
            delete modKeysRef.value[orgKeyInfo?._ROW_ID]
          })
          .catch(error => failList.push({ item: dataItem, error: error }))
          .finally(() => promises_race.splice(promises_race.indexOf(promise), 1))
        promises_race.push(promise)
        promises_all.push(promise)
      } else {
        passList.push(dataItem)
      }
    }
    await Promise.all(promises_all)

    let message = ''
    const passCount = passList.length
    const failCount = failList.length
    if (passCount) {
      message += `Successfully deleted(${passCount}).`
      grid.applyTransaction({ remove: passList })
      onRefreshCells()

      gridDataRef.value = gridDataRef.value.filter(({ _ROW_ID }) => {
        return !passList.some(v => v._ROW_ID === _ROW_ID)
      })
      gridDataOrgRef.value = gridDataOrgRef.value.filter(({ _ROW_ID }) => {
        return !passList.some(v => v._ROW_ID === _ROW_ID)
      })
    }
    if (failCount) {
      if (!message) {
        message += `Failed deleted.`
      }
      message += `\nfailed to delete ${failCount} item.`
      for (let i = 0; i < failCount; i++) {
        const { item, error } = failList[i]
        const backMsg = String(error.response?.data?.Message || error.message).replace(/\r\n/g, '<br>')
        message += `\n    [ Register ID : ${item.TYPE_ID || ''}, Reason: ${backMsg} ]`
      }
    }
    if (!passCount && failCount) {
      toast.fail({ okText: '확인', title: message })
    } else {
      toast.success({ title: message, duration: 4000 })
    }
    setFocusData()
    return onFinally()
  }

  const checkValidation = function (dataList) {
    const focusData = focusDataRef.value
    const isSubEdited = isSubEditedRef.value

    const dupIdCheck = {}
    const failMsgList = []
    dataList?.forEach(dataItem => {
      const regTypeId = dataItem.TYPE_ID
      const desc = dataItem.DESC
      const epId = dataItem.EP_ID
      const projNo = dataItem.PROJ_NO
      const newTagYn = dataItem.NEW_TAG_YN
      const rowId = dataItem._ROW_ID

      if (regTypeId) {
        dupIdCheck[regTypeId] = !dupIdCheck[regTypeId]
      }

      if (!regTypeId) {
        failMsgList.push(`'Register ID' is Required.`)
      } else if (!dupIdCheck[regTypeId]) {
        failMsgList.push(`[${regTypeId}] 'Register ID' is Duplicated.`)
      } else if (!projNo) {
        failMsgList.push(`[${regTypeId}] 'Project' is Required.`)
      } else if (!epId) {
        failMsgList.push(`[${regTypeId}] 'EP' is Required.`)
      } else if (!desc) {
        failMsgList.push(`[${regTypeId}] 'Description' is Required.`)
      } else if (
        !isSetupAdmin.value &&
        !projectPermissions.value?.some(v => v.EP_ID === epId && v.Role === 'Manager')
      ) {
        failMsgList.push(`[${regTypeId}] Do not have permission to 'EP'[${epId}].`)
      } else {
        if (focusData?._ROW_ID === rowId && isSubEdited) {
          let isSubFailed = false
          if (!isSubFailed) {
            // MAP_REL - 필수 값만 확인
            const relFailed = mapRelDataListRef.value?.find(v => !v.REL_ID || !v.REG_TYPE_ID)
            if (relFailed) {
              isSubFailed = true
              const { REL_ID, REG_TYPE_ID } = relFailed
              failMsgList.push(
                `[${regTypeId}] ${
                  !REL_ID ? 'Relation' : !REG_TYPE_ID ? 'Register' : ''
                } in the 'Relation Map' tab is Required.`
              )
            }
          }
          if (!isSubFailed) {
            // JOIN_TABLE - 필수 값만 확인
            const joinFailed = joinTableDataListRef.value?.find(v => !v.ALIAS || !v.TYPE || !v.TABL_ID)
            if (joinFailed) {
              isSubFailed = true
              const { ALIAS, TYPE, TABL_ID } = joinFailed
              failMsgList.push(
                `[${regTypeId}] ${
                  !ALIAS ? 'Table Alias' : !TYPE ? 'Table Type' : !TABL_ID ? 'Table ID' : ''
                } in the 'Join Table' tab is Required.`
              )
            }
          }
          if (!isSubFailed) {
            // MAP_CLS - 2개 이상 ROOT_CLASS 확인
            let rootClsId
            mapClsDataListRef.value?.forEach(v => {
              if (!isSubFailed && v._IS_SELECTED) {
                const newRootClsId = v.path?.[0]
                if (!rootClsId) {
                  rootClsId = newRootClsId
                } else if (rootClsId !== newRootClsId) {
                  isSubFailed = true
                }
              }
            })
            if (isSubFailed) {
              failMsgList.push(`[${regTypeId}] 'Class Map' cannot hav different root classes.`)
            }
       
