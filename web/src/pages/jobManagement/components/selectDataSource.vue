<template>
  <div class="sds-wrap">
    <!-- <a-button type="dashed" @click="showModal">{{ defaultSelect }}</a-button> -->
    <div
      class="sds-button"
      @click="showModal"
      v-if="defaultSelect === '请点击后选择'"
    >
      <PlusOutlined
        style="color: rgba(0, 0, 0, 0.65); font-size: 12px; margin-right: 8px"
      />
      <span>{{ defaultSelect }}</span>
    </div>
    <div v-else class="sds-title-tags">
      <div class="sds-title-tag"
           v-for="(item, idx) in defaultSelect"
           :key="idx"
           @click="showModal"
           :title="item"
      >
        <span v-if="idx===0"><span class="logo" :style="getBg()"> </span> {{ item }}</span>
        <span v-else>{{ item }}</span>
      </div>
    </div>
    <a-modal
      v-model:visible="visible"
      title="选择数据源"
      @ok="handleOk"
      width="500px"
    >
      <!-- top -->
      <div class="sds-wrap-t">
        <a-space size="middle">
          <span>数据类型</span>
          <a-select
            v-model:value="curSql"
            style="width: 150px"
            placeholder="请先选择数据库"
            :options="
              sqlList.map((sql) => ({ value: sql.value, label: sql.name }))
            "
            @change="handleChangeSql"
          >
          </a-select>
          <span>数据源</span>
          <a-select
            placeholder="请先选择数据源"
            style="width: 150px"
            v-model:value="dataSource"
            @change="handleChangeDS"
            :options="
              dataSourceList.map((ds) => ({ value: ds.value, label: ds.name }))
            "
          ></a-select>
        </a-space>
        <a-space size="middle" v-if="dataSource">
          <span>搜索库</span>
          <a-input
            placeholder="按回车搜库"
            style="width: 300px;margin-top: 10px;margin-left:14px"
            v-model:value="searchDB"
            @keyup.enter="showTableSearch=false;createTree(dsId)"
          ></a-input>
        </a-space>
        <a-space size="middle" v-if="dataSource && showTableSearch">
          <span>搜索表</span>
          <a-input
            placeholder="按回车搜表"
            style="width: 300px;margin-top: 10px;margin-left:14px"
            v-model:value="searchWord"
            @keyup.enter="filterTree(dsId)"
          ></a-input>
        </a-space>
      </div>
      <!-- bottom 类似tree组件 -->
      <div class="sds-wrap-b">
        <a-spin :spinning="spinning" style="
          text-align: center;
          width: 100%;
          height: 300px;
          padding-top: 150px;"/>
        <a-directory-tree
          :height="400"
          :tree-data="treeData"
          :autoExpandParent="false"
          v-model:expandedKeys="expandedKeys"
          @select="selectItem"
          @expand="handleExpandSql"
        >
        </a-directory-tree>
      </div>
    </a-modal>
  </div>
</template>

<script>
import { PlusOutlined, MinusOutlined } from "@ant-design/icons-vue";
import {
  getDataSourceTypes,
  getDBs,
  getTables,
  getDataSource,
} from "@/common/service";
import {
  defineComponent,
  reactive,
  toRefs,
  computed,
  watch,
  ref,
  toRaw,
  onMounted,
} from "vue";
import { message } from "ant-design-vue";

export default defineComponent({
  props: {
    title: [Object, String],
  },
  components: {
    PlusOutlined,
    MinusOutlined,
  },
  emits: ["updateDsInfo"],
  setup(props, context) {
    let SQLlist, treeData, originTreeData;
    const sqlList = [];
    const expandedKeys = ref([]);
    const state = reactive({
      sqlSource: sqlList.length ? sqlList[0].value : "",
      sqlList,
      defaultSelect: props.title,
      treeData,
      originTreeData,
      sqlId: "",
      dsId: "",
      curSql: "",
      dataSource: "",
      dataSourceList: [],
      selectTable: "",
      searchWord: '',
      searchDB: ''
    });
    let spinning = ref(false)
    const newProps = computed(() => JSON.parse(JSON.stringify(props.title)));
    watch(newProps, (val, oldVal) => {
      state.defaultSelect = val;
    });
    async function init() {
      SQLlist = (await getDataSourceTypes()).list;
      // 数据源
      SQLlist.forEach((sql) => {
        sqlList.push({
          name: sql.name,
          value: sql.name,
          id: sql.id,
        });
      });
      if (state.sqlSource) await createTree(state.sqlSource);
    }
    onMounted(init());
    // 根据数据类型ID的不同返回不同的数据源列表
    const queryDataSource = async (typeId) => {
      // 这里前端目前不做分页 固定了 page 和 pageSize
      let body = {
        name: "",
        typeId,
        page: 1,
        pageSize: 1000,
      };
      let res = [];
      try {
        let _res = await getDataSource(body);
        const { list } = _res;
        list.forEach((item) => {
          let o = Object.create(null);
          o.name = item.name;
          o.value = item.id;
          res.push(o);
        });
      } catch (err) {
        console.log("err");
      }
      return res;
    };

    let showTableSearch = ref(false)

    // 选择数据库触发
    const handleChangeSql = async (sql) => {
      state.curSql = sql;
      const cur = state.sqlList.filter((i) => i.value === sql)[0];
      state.sqlId = cur.id;
      let dsOptions = await queryDataSource(cur.id);
      state.dataSourceList = dsOptions.length > 0 ? dsOptions : [];
      // 清空
      state.dataSource = ''
      state.dsId = ''
      state.searchWord = ''
      state.searchDB = ''
      state.treeData = []
      state.originTreeData = []
      showTableSearch.value = false
    };
    // 选择数据源触发
    const handleChangeDS = async (ds) => {
      // 清空
      state.searchWord = ''
      state.searchDB = ''
      showTableSearch.value = false
      createTree(ds, () => {
        const cur = state.dataSourceList.filter((item) => {
          return item.value === ds;
        })[0];
        state.dataSource = cur.name;
        state.dsId = cur.value;
      });
    };
    // 创建 db & tables tree
    const createTree = async (ds, cb) => {
      if (!ds) return;
      spinning.value = true;
      const tree = [];
      // 这里 根据数据源请求 dbs
      const cur = state.dataSourceList.filter((item) => {
        return item.value === ds;
      })[0];
      let dbs
      try {
        dbs = (await getDBs(state.curSql, cur.value)).dbs;
      } catch (e) {
        dbs = []
        console.log(e)
      }
      if (state.searchDB) {
        dbs = dbs.filter((db) => {
          return new RegExp(state.searchDB).test(db)
        })
        showTableSearch.value = true
      }
      if (!dbs) return;
      dbs.forEach((db, index) => {
        const o = Object.create(null);
        o.selectable = false;
        o.title = db;
        o.key = db;
        o.children = [];
        // const oo = Object.create(null);
        // (oo.key = ""), (oo.title = ""), o.children.push(oo);
        tree.push(o);
      });
      state.treeData = tree;
      state.originTreeData = [].concat(tree); // 必须保存一份原始数据，用于过滤库表
      spinning.value = false;
      cb && cb();
    };
    // filter tree
    const filterTree = (ds) => {
      if (!state.searchDB) {
        return message.error('请确定搜索的库')
      }
      // 直接从originTreeData对库和表进行过滤
      const tree = toRaw(state.originTreeData);
      if (state.searchWord) {
        /*state.treeData = tree.filter(i => {
          return new RegExp(state.searchWord, "i").test(i.title) || i.children.find(c => new RegExp(state.searchWord, "i").test(c.title));
        }).map(i => {
          return {
            ...i,
            children: i.children.filter(c => new RegExp(state.searchWord, "i").test(c.title))
          }
        });*/
        state.treeData = tree.map(i => {
          return {
            ...i,
            children: i.children.filter(c => new RegExp(state.searchWord, "i").test(c.title))
          }
        })

      } else {
        createTree(ds)
      }
    };
    const visible = ref(false);
    const showModal = () => {
      visible.value = true;
    };
    const selectItem = (e) => {
      state.selectTable = e.join("");
      // state.defaultSelect = `${state.sqlSource}-${state.dataSource}-${e.join(
      //   ""
      // )}`;
    };
    const handleOk = () => {
      if (!state.curSql) {
        return message.error("未正确选择数据类型");
      }
      if (!state.dataSource) {
        return message.error("未正确选择数据源");
      }
      if (!state.selectTable) {
        return message.error("未正确选择库表");
      }
      state.defaultSelect = `${state.curSql}.${state.dataSource}.${state.selectTable}`;
      visible.value = false;
      context.emit("updateDsInfo", state.defaultSelect, state.dsId);
      // 选择完 初始化数据
      state.curSql = "";
      state.dataSource = "";
      state.selectTable = "";
      state.treeData = [];
      state.originTreeData = [];
      expandedKeys.value = [];
    };
    /**
     * 获取数据库下 所有表
     * @param (sql, dsId, dbName)
     * @returns tbales
     */
    const asyncGetTables = async (sql, dsId, dbName) => {
      let tables;
      const res = [];
      tables = (await getTables(sql, dsId, dbName)).tbs || [];
      tables.forEach((table) => {
        const o = Object.create(null);
        o.title = table;
        o.key = `${dbName}.${table}`;
        res.push(o);
      });
      return res;
    };
    // 展开数据库树获取表叶子
    const handleExpandSql = async (expandedKeys, { expanded, node }) => {
      const dbName = node.title;
      const loaded = state.originTreeData.find(i => i.title == dbName && i.children.length > 0);
      if (!loaded) {
        let tables = await asyncGetTables(state.curSql, state.dsId, dbName);
        state.treeData.forEach(async (td) => {
          if (td.title === dbName) {
            // 应该根据searchWord过滤，不然用户先搜索，然后在展开库，仍然会看到所有表，体验不好
            if (state.searchWord) {
              tables = tables.filter(i => new RegExp(state.searchWord, "i").test(i.title))
            }
            tables.forEach(tb => {
              tb.isLeaf = true;
            })
            return (td.children = tables.slice());
          }
        });
        state.originTreeData.forEach(async (td) => {
          if (td.title === dbName) {
            tables.forEach(tb => {
              tb.isLeaf = true;
            })
            return (td.children = tables.slice());
          }
        });
      }
    };
    const getBg = () => {
      let name = state.curSql || (typeof state.defaultSelect === 'string' ? state.defaultSelect.split('.')[0] : state.defaultSelect[0])
      return `background-image: url(${require('@/images/dataSourceTypeIcon/' + name + '.png')})`
    }
    return {
      ...toRefs(state),
      selectItem,
      visible,
      showModal,
      handleOk,
      sqlList,
      handleChangeSql,
      handleExpandSql,
      handleChangeDS,
      getBg,
      expandedKeys,
      createTree,
      filterTree,
      spinning,
      showTableSearch
    };
  },
});
</script>

<style lang="less" scoped>
.sds-wrap {
  display: inline-block;
  .logo {
    width: 20px;
    height: 20px;
    float: left;
    background-size: cover;
  }
  .sds-button {
    width: 420px;
    height: 46px;
    background: #f8fafd;
    border: 1px dashed #dee4ec;
    border-radius: 2px;
    line-height: 46px;
    text-align: center;

    font-family: PingFangSC-Medium;
    font-size: 14px;
    color: rgba(0, 0, 0, 0.65);
    font-weight: 500;
    cursor: pointer;
  }

  .sds-title-tags {
    width: 420px;
    height: 46px;
    display: flex;
    /*justify-content: center;*/
    align-items: center;
    .sds-title-tag {
      overflow: hidden;
      white-space: nowrap;
      text-overflow: ellipsis;
      padding: 5px 15px;
      background: #e6f0ff;
      font-family: PingFangSC-Medium;
      font-size: 14px;
      color: rgba(0, 0, 0, 0.65);
      font-weight: 500;
      height: 32px;
      min-width: 100px;
      max-width: 100px;
      cursor: pointer;
      text-align: center;
      border-top: 1px solid #dee4ec;
      border-bottom: 1px solid #dee4ec;
      border-right: 1px solid #dee4ec;
      &:first-child {
        border-left: 1px solid #dee4ec;
        border-radius: 4px 0 0 4px;
      }
      &:last-child {
        border-radius: 0 4px 4px 0;
      }
    }
    &:hover {
      .sds-title-tag {
        color: #2E92F7;
      }
    }
  }
}
.sds-wrap-b {
  min-height: 300px;
  max-height: 400px;
  overflow-y: auto;
}

</style>
