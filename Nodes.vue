<script>
import { reactive, toRefs, inject, onActivated, onMounted, watch, computed, onUnmounted, } from "vue";
import { useRoute } from "vue-router";
import Node from "./Node.vue";
import LeaderLine from "leader-line-vue";
import UploadList from "./UploadList.vue";
import WildcardTable from "./WildcardTable.vue"
import Empty from "./layer/Empty.vue"
import PipeInfo from "./PipeInfo.vue";
import { pickColor, isScrollShow, unique, getRelativePosition, getLineOptions, } from "../utils/utils";
import { useStore } from "vuex";
import { useI18n } from "vue-i18n";
const md5 = require("md5");
const DEFAULT_POSITION = '1,0,0,1,0,0';
const LISE_BASE_SIZE = 4;
const RADIAN = 120;
export default {
  components: { Node, UploadList, WildcardTable, Empty, PipeInfo },
  props: {
    groupId: Number,
    group: Object,
  },
  emits: ["deleteNode", "update", 'restoreLayout'],
  setup(props, { emit }) {
    const { $axios, $notify } = inject("$");
    const route = useRoute()
    const store = useStore()
    const { t } = useI18n()
    const v = reactive({
      nodes: [],
      loading: false,
      loadnodesing: false,
      isEmpty: false,
      closeDelete: null,
      closePath: null,
      closeName: null,
      closeDeletePath: null,
      closeDeleteDisk: null,
      closeDeltePipeModal: null,
      closeConfimSave: null,
      tobeDelete: { group: props.groupId },
      tobeaddNodeId: null,
      tobeaddIp: null,
      tobeaddMac: null,
      tobeaddMountpoint: null,
      newPath: null,
      newName: null,
      oldName: null,
      newPathErr: false,
      linesList: [],
      poddList: [],
      deleteSource: false,
      openMoveFileModal: null,
      openUpdateFileModal: null,
      closeMoveFileModal: null,
      tobeUpdatePipe: {},
      pipes: [],
      tobeDeletePipe: [],
      moveFile: { force_over_write: false, filters_type: 'exclude', sync_type: 'relative' },
      onStartParams: { id: null, path: null },
      saveing: null,
      filters: [],
      newFilter: null,
      scale: 1,
      scaleChange: false,
      transform: props.group?.transform || localStorage.getItem('_transform') || DEFAULT_POSITION,
    });

    watch(() => props.groupId, () => {
      for (let key in lines) {
        lines[key]?.remove(); // 清理旧线
        delete lines[key]
      }
      v.nodes = []
      v.transform = props.group.transform
      document.querySelector('#zoomMe').style.transform = `matrix(${v.transform})`;
      view = createView()
      getNodes()
    })

    watch(() => route.path, () => {
      for (let key in lines) {
        lines[key]?.remove();
        delete lines[key]
      }
    })

    watch(() => v.scale, () => {
      if (v.scaleChange) clearTimeout(v.scaleChange)
      v.scaleChange = setTimeout(() => { v.scaleChange = false }, 2000)
    })
    let lines = {};

    async function getNodes() {
      try {
        v.loadnodesing = true
        const res = await $axios.get('/api/v1/nodes', { id: props.groupId })
        if (res) {
          let list = [];
          res.data.forEach((node, id) => {
            if (!node.left) {
              let { left, top } = getNewNodePosition(node, res.data)
              node.left = left
              node.top = top
            }
            node.disks?.forEach((e, i) => {
              e.pathList &&
                e.pathList.forEach((item, index) => {
                  list.push(id + "path" + i + "-" + index);
                });
            });
          });

          await getPipe()
          v.nodes = res.data;
          v.poddList = list;

          if (v.nodes.length >= 1) {
            let maxZ = v.nodes.reduce((prev, current) => (prev.zIndex > current.zIndex ? prev : current));
            store.commit('SET_ZINDEX', maxZ.zIndex)
            if (maxZ.zIndex > 9000) {
              let zlist = res.data.map(({ id, zIndex }) => { return { id, zIndex } })
              zlist.sort((x, y) => (x.zIndex > y.zIndex) ? 1 : -1)

              zlist.forEach((e, i) => {
                v.nodes.forEach(node => { if (node.id == e.id) node.zIndex = i })
              })
              confirmSave()
            }
          }
          nodedata()
          setTimeout(() => {
            createlineList(v.pipes);
          }, 200)
        } else {
          v.nodes = []
        }
      } catch (error) {
        console.error(error);
      } finally {
        v.loadnodesing = false
        v.isEmpty = v.nodes.length == 0
      }
    }

    async function getPipe() {
      if (!props.groupId) return
      const res = await $axios.get("/api/v1/sync", { group: props.groupId });
      if (res.data) {
        v.pipes = res.data;
      }
    }


    function getNewNodePosition(node, nodes) {
      // 检查是否有node没有位置

      const width = 500 * v.scale
      const height = 509 * v.scale
      const SPACE = 100 * v.scale
      const MARGIM_TOP = 21
      const VW = document.body.clientWidth
      // node.saveLeft = true
      // 直接找最下面的 计算右侧剩余空间    
      if (nodes.length == 0) {
        node.left = '21px'
        node.top = '61px'
        return node
      }
      let maxTop = nodes.reduce((prev, current) => (Math.ceil(prev.top.replace('px', '')) > Math.ceil(current.top.replace('px', ''))) ? prev : current);
      if (!maxTop || maxTop.id == node.id) {
        node.left = '21px'
        node.top = '61px'
      } else {
        let newleft;
        let newtop; //= Number(maxTop.top.replace('px', '')) + height + 10 + 'px'
        let surplus = VW - (Number(maxTop.left.replace('px', '')) + width)
        if (surplus > (width + SPACE)) {
          newleft = Number(maxTop.left.replace('px', '')) + width + SPACE + 'px'
          newtop = maxTop.top
          console.log(surplus);
        } else {
          newleft = MARGIM_TOP + 'px'
          newtop = Number(maxTop.top.replace('px', '')) + height + MARGIM_TOP + 'px'
          console.log('have no sapce');
        }
        node.left = newleft
        node.top = newtop
      }
      console.log(node.left, node.top);
      return node
    }

    async function updateNode(data) {
      try {
        v.loadnodesing = true
        const res = await $axios.get('/api/v1/node', {
          id: data.node_id, group: props.groupId
        })
        const node = res.data
        let index = v.nodes.findIndex(e => e.id == data.node_id)
        console.log(node.left, index);

        if (!node.left) {
          let { left, top } = index < 0 ? getNewNodePosition(node, v.nodes) : v.nodes[index]
          node.left = left
          node.top = top
        }

        if (node.group == props.groupId) { //防止出现获取到node时group已经被修改
          if (index >= 0) {
            v.nodes.splice(index, 1, res.data)
          } else {
            v.nodes.push(node)
          }
          setTimeout(() => { linesPosition() }, 200)
          v.isEmpty = v.nodes.length == 0
          nodedata()
        }
      }
      catch (error) { console.error(error) }
      finally { v.loadnodesing = false }
    }

    async function deleteNode() {
      let obj = { id: v.tobeDelete.id, group: props.groupId, };
      try {
        v.loading = true;
        const res = await $axios.delete("/api/v1/node", obj);
        if (res.status == 200) {
          getNodes()
          v.closeDelete.click();
          $notify({ type: "success", title: t('notify.reqSuccess') });
        }
      } catch (error) {
        console.error(error);
      } finally {
        v.loading = false;
      }
    }

    async function createPath() {
      try {
        v.loading = true;
        await $axios.post("/api/v1/path", {
          ip: v.tobeaddIp, mountpoint: v.tobeaddMountpoint, node_id: v.tobeaddNodeId,
          path: v.newPath, group: props.groupId,
        });
        v.closePath.click();
        getNodes()
      } catch (error) {
        console.error(error)
      } finally { v.loading = false }
    }

    async function putName() {
      try {
        v.loading = true;
        const res = await $axios.put("/api/v1/nodeName", {
          id: v.tobeaddNodeId,
          name: v.newName,
          group: props.groupId,
        });

        if (res.status == 200) {
          v.nodes.forEach(node => {
            if (node.id == v.tobeaddNodeId) {
              node.name = v.newName
            }
          })
          v.newName = "";
          $notify({ type: "success", text: t('notify.upSuccess') });
          v.closeName.click()
        }
      } catch (error) {
        console.error(error);
      } finally {
        v.loading = false;
      }
    }

    async function deleteDisk() {
      let requestPath = "/api/v1/mountpoint";
      v.tobeDelete.group = props.groupId;
      try {
        v.loading = true;
        const res = await $axios.delete(requestPath, v.tobeDelete);
        if (res.status == 200) {
          v.tobeDelete = { group: props.groupId };
          v.closeDeleteDisk.click();
          getNodes()
        }
      } catch (error) {
        console.error(error);
      } finally {
        v.loading = false;
      }
    }

    async function deletePath() {
      v.tobeDelete.group = props.groupId;
      let requestPath = v.deleteSource ? "/api/v1/pathAndDir" : "/api/v1/path";
      try {
        v.loading = true;
        const res = await $axios.delete(requestPath, v.tobeDelete);
        if (res.status == 200) {
          // 本地删除路径
          let node = v.nodes.find(e => e.id == v.tobeDelete.node_id)
          let disk = node.disks.find(d => d.mountpoint == v.tobeDelete.mountpoint)
          disk.paths = disk.paths.filter(e => e.path != v.tobeDelete.path)
          await getPipe()
          setTimeout(() => {
            createlineList(v.pipes)
          }, 200)
          v.tobeDelete = {};
          v.closeDeletePath.click();
        }
      } catch (error) {
        console.error(error);
        $notify({ type: "error", text: error });
      } finally {
        v.loading = false;
      }
    }

    async function deletePipe() {
      try {
        let ids = v.tobeDeletePipe.filter(e => e.checked).map(e => e.id)
        v.loading = true;
        const res = await $axios.delete("/api/v1/syncs", {
          group: props.groupId,
          ids,
        });
        if (res.status == 200) {
          ids.forEach((item) => {
            v.pipes = v.pipes.filter(e => e.id != item)
          })
          createlineList(v.pipes)
          nodedata()
          handelCloseDeletePipeModal()
          v.closeDeltePipeModal.click();
        }
      } finally {
        v.loading = false;
      }
    }

    async function handelMoveFile() {
      let { startId, endId, type, to_ip } = v.moveFile;
      console.log(startId, endId, type);
      delete v.moveFile.startId;
      delete v.moveFile.endId;
      delete v.moveFile.type;
      v.moveFile.filters = v.filters
      try {
        v.loading = true;
        const res = await $axios.post("/api/v1/sync", v.moveFile);
        $notify({ type: "info", text: t('notify.pleaseWait') });
        let lastId = Math.max(...v.pipes.map((e) => e.id));
        v.linesList.push(startId, endId, type, pickColor(lastId + 1 + ""));
        v.closeMoveFileModal.click();
        // $notify({ type: "success", title: t('') });
        v.moveFile = { force_over_write: false, filters_type: 'exclude', sync_type: 'relative' };
        v.filters = []
        await getPipe();
        nodedata()
        setTimeout(() => {
          createlineList(v.pipes);
        }, 100)

      } catch (error) {
        console.error(error);

      } finally {
        v.loading = false;
      }
    }

    async function updatePosition() {
      try {
        v.saveing = true
        await $axios.put('/api/v1/groupSize', {
          id: props.groupId, transform: v.transform
        })
        emit('update', v.transform)
      } catch (error) {
        console.error(error);
      } finally {
        v.saveing = false
        store.commit('SET_DRAP', false)
      }
    }

    async function confirmSave() {
      try {
        v.saveing = true
        let nodes = []
        v.nodes.forEach(({ id, top, left, zIndex }) => {
          nodes.push({ id, top, left, zIndex })
        })
        await $axios.put('/api/v1/nodes', { group: props.groupId, nodes })
        await updatePosition()
        v.closeConfimSave.click()
      } catch (error) {
        console.error(error);
      } finally {
        v.saveing = false
      }
    }

    function nodedata() {
      let nodes = v.nodes?.length || 0;
      let paths = 0;
      v.nodes?.forEach((e) => {
        e.disks?.forEach((d) => {
          paths += d.paths?.length || 0;
        });
      });
      store.commit('SET_NODEDATA', { nodes, paths, pipes: v.pipes.length, ErrorPipe: 0 })
    };

    // 生成拉线数组
    function createlineList(pipes) {
      let list = [];
      if (!v.nodes) return;
      if (!pipes) pipes = v.pipes
      const nodes = v.nodes;

      pipes.forEach((item) => {
        let fromIpIndex = nodes.findIndex(
          (e) => e.ip == item.from_ip.split(":")[0]
        );

        if (fromIpIndex < 0) return console.log(item.from_ip, fromIpIndex);

        let fromDisk = nodes[fromIpIndex].disks.find((e) =>
          e.paths?.find(p => p.path == item.sources[0])
        );

        if (fromIpIndex < 0 || fromDisk.id <= 0) return console.log(fromIpIndex, fromDisk);

        let fromPath = fromDisk.paths.find((e) => e.path == item.sources[0]);

        if (!fromPath) return console.log(fromDisk)

        let from = fromIpIndex + "-" + fromDisk.id + "-" + fromPath.id;

        let toIpIndex = nodes.findIndex((e) => e.ip == item.to_ip.split(":")[0]);
        if (toIpIndex < 0) return console.log(toIpIndex);

        let toDisk = nodes[toIpIndex].disks.find((e) =>
          e.paths?.find(p => p.path == item.destination)
        );

        if (toDisk.id <= 0) return console.log(toDisk);

        let toPath = toDisk.paths.find((e) => e.path == item.destination);


        if (toIpIndex < 0 || toPath.id <= 0) return console.log('return', toIpIndex, toPathIndex);
        let to = toIpIndex + "-" + toDisk.id + "-" + toPath.id;

        let position = getRelativePosition(
          "node" + fromIpIndex,
          "node" + toIpIndex
        );

        from = from + position.start;
        to = to + position.end;

        // 这里判断前面是否有同位置线 尽量不重叠 多于7条才重叠
        let fromcount = list.filter((e) => e.from.indexOf(from) >= 0 || e.to.indexOf(from) >= 0).length;
        let tocount = list.filter((e) => e.to.indexOf(to) >= 0 || e.from.indexOf(to) >= 0).length;

        if (fromcount >= 7) fromcount = fromcount % 7;
        if (tocount >= 7) tocount = tocount % 7;


        from = from + (fromcount + 1);
        to = to + (tocount + 1);

        // type 1和3相反 2和4相反
        let type = 2;

        if (position.start == position.end && position.start == 'right') {
          type = 4
        }

        if (position.start == 'left' && position.end == 'right') {
          type = 3
        }
        if (position.start == 'right' && position.end == 'left') {
          type = 1
        }

        console.log(from, to, type);

        list.push({
          from, to, type,
          color: pickColor(md5(item.id)),
          id_from: item.id,
          hide: false,
        });
      });

      v.linesList = unique(list);

      if (document.querySelector(".podd")) reloadline();

    }

    // 画线的方法
    function line(startId, endId, type, color, id_from, hide) {
      const size = v.scale
      let line = LeaderLine.setLine(
        document.getElementById(startId),
        document.getElementById(endId),
        { dash: { animation: false }, color, size: LISE_BASE_SIZE * size, dropShadow: true, opacity: 0.1, hide }
      );
      line.id_from = id_from;
      line.lineType = type;
      line.setOptions(getLineOptions(type, RADIAN, size))
      return line;
    }

    function reloadline() {
      for (let key in lines) {
        lines[key]?.remove(); // 清理旧线
      }
      lines = {}
      v.linesList.forEach((e, i) => {
        if (!e) return
        let from = e.from
        let to = e.to

        let fromPosition = isScrollShow(e.from)
        let toPosition = isScrollShow(e.to)


        if (!fromPosition.show) {
          let leftOrRight = (e.type == 4 || e.type == 1) ? 'right' : 'left'
          let topOrBottom = fromPosition.up ? 't' : 'b'
          from = `node-${leftOrRight}-${topOrBottom}-s${e.from.split('-')[0]}`
        }
        if (!toPosition.show) {
          let leftOrRight = (e.type == 4 || e.type == 3) ? 'right' : 'left'
          let topOrBottom = toPosition.up ? 't' : 'b'
          to = `node-${leftOrRight}-${topOrBottom}-e${e.to.split('-')[0]}`
        }
        let newline = line(from, to, e.type, e.color, e.id_from, e.hide);
        lines[e.from + "~" + e.to] = newline

        e._id = newline._id

        let line_use = document.querySelector(`#leader-line-${e._id}-line-path`)
        const line_svg = line_use.parentElement.parentElement //.getElementsByTagName('g')[3]
        let pipe = v.pipes.find(p => p.id == e.id_from)

        let fromPathId = e.from.replace("-", "path").replace("right", "").replace("left", "")
          .split('').slice(0, -1).join('');
        let toPathId = e.to.replace("-", "path").replace("right", "").replace("left", "")
          .split('').slice(0, -1).join('');

        line_svg.addEventListener("mouseover", function () {
          if (v.scale < 0.5) return
          let path = document.querySelector(`#leader-line-${e._id}-line-path`)
          let lineSvg = path.parentNode.parentNode
          lineSvg.style['z-index'] = '9999'
          lines[e.from + "~" + e.to].setOptions({
            dropShadow: { color: '#fff', dx: 0, dy: 0 }
          })
          // document.getElementById(fromPathId).style.borderColor = "#f38506";
          // document.getElementById(toPathId).style.borderColor = "#f38506";

        });

        line_svg.addEventListener("mouseout", function () {
          if (v.scale < 0.5) return
          let path = document.querySelector(`#leader-line-${e._id}-line-path`)
          let lineSvg = path.parentNode.parentNode
          lineSvg.style['z-index'] = ''
          lines[e.from + "~" + e.to].setOptions({
            size: LISE_BASE_SIZE * v.scale,
            dropShadow: {}
          })
          // document.getElementById(fromPathId).style.borderColor = "";
          // document.getElementById(toPathId).style.borderColor = "";

        });

        line_svg.addEventListener('click', function (e) {
          line_svg.style['z-index'] = ''
          v.tobeUpdatePipe = JSON.parse(JSON.stringify(pipe))
          v.openUpdateFileModal.click()
        })

      });

    }

    function handleScrollReloadLine(i) { // i是node的index
      // reloadline()
      for (let key in lines) {
        // 首选判断和这个条线有没有关系
        let list = key.split('~')
        let from = list[0]
        let to = list[1]
        let lineOptions = lines[key]
        if (!lineOptions) return;
        let type = lineOptions.lineType

        if (from.indexOf(i) == 0 || to.indexOf(i) == 0) {
          /*
         //总共有几种情况 
         1.俩都在显示 就正常显示就行了
         2.一个在一个不在 最难是这个一半一半的 然后就需要知道是来自上面还是来自下面
         3.俩都不在 ； 直接不用显示这个线了
         4.先是消失了然后又显示了
         */

          let fromPosition = isScrollShow(from)
          let toPosition = isScrollShow(to)

          // if (!fromPosition.show && !toPosition.show) {
          //   lineOptions.hide();
          //   continue;
          // }
          lineOptions.show();
          if (!fromPosition.show) {
            let leftOrRight = (type == 4 || type == 1) ? 'right' : 'left'
            let topOrBottom = fromPosition.up ? 't' : 'b'
            from = `node-${leftOrRight}-${topOrBottom}-s${from.split('-')[0]}`
            lineOptions.setOptions({
              start: document.getElementById(from)
            })
          } else {
            if (lineOptions.start.id != from) {
              lineOptions.setOptions({
                start: document.getElementById(from)
              })
            }
          }

          if (!toPosition.show) {
            let leftOrRight = (type == 4 || type == 3) ? 'right' : 'left'
            let topOrBottom = toPosition.up ? 't' : 'b'
            to = `node-${leftOrRight}-${topOrBottom}-e${to.split('-')[0]}`
            lineOptions.setOptions({
              end: document.getElementById(to)
            })
          } else {
            if (lineOptions.end.id != to) {
              lineOptions.setOptions({
                end: document.getElementById(to)
              })
            }
          }

          if (fromPosition.show && toPosition.show) {

            lineOptions.setOptions({
              start: document.getElementById(from),
              end: document.getElementById(to)
            })
          }
          lineOptions?.position();
        }

      }
    }

    function handelenter(linename) {
      for (let key in lines) {
        if (key.indexOf(linename) >= 0) {
          const line = lines[key];
          line.oldColor = line.color;
          // line.color = "#f38506";
          //line.color = line.color.replace(', 0.3)', ')');
          //  line.size = 6 * v.scale;
          line.dropShadow = { color: '#fff', dx: 0, dy: 0 }
          let path = document.querySelector(`#leader-line-${line._id}-line-path`)
          let lineSvg = path.parentNode.parentNode
          lineSvg.style['z-index'] = '9999'
        }
      }

      let targetlist = findTheOther(linename);
      targetlist.forEach((e) => {
        let id = e.replace("-", "path").replace("right", "").replace("left", "");
        id = id.substring(0, id.length - 1);
        document.getElementById(id).style.borderColor = "#f38506";
      });
    }

    function handleleave(linename) {
      for (let key in lines) {
        if (key.indexOf(linename) >= 0) {
          const line = lines[key];
          line.color = line.oldColor;
          line.size = LISE_BASE_SIZE * v.scale;
          line.dropShadow = {}
          let path = document.querySelector(`#leader-line-${line._id}-line-path`)
          let lineSvg = path.parentNode.parentNode
          lineSvg.style['z-index'] = ''
        }
      }
      // 出现了因为网络延长问题导致的 触发事件时已经没有对应的线条了 所以改成出框直接全清
      document.querySelectorAll('.path').forEach(e => e.style.borderColor = 'transparent')
    }

    function handelMove(end) {
      end ? createlineList(v.pipes) : linesPosition()
    }

    function handelCloseDeletePipeModal() {
      v.tobeDeletePipe = []
    }


    function handleCreateLine(params) {
      let { startId, endId } = params;
      lines[startId + "~" + endId] = LeaderLine.setLine(
        document.getElementById(startId),
        document.getElementById(endId),
        {
          dash: { animation: true },
          color: "#f38506",
          size: 8,
        }
      );
    }

    function findTheOther(param) {
      let list = [];
      for (let key in lines) {
        if (key.indexOf(param) >= 0) {
          list.push(key);
        }
      }

      let returnlist = [];
      list.forEach((e) => {
        let from = e.split("~")[0];
        let to = e.split("~")[1];
        returnlist.push(to);
        returnlist.push(from);
      });

      return returnlist;
    }

    function handleDragEnd({ endId, endPath }) {
      v.onStartParams.endId = endId
      if (!v.onStartParams.id) return
      let startId = v.onStartParams.id.replace('shadow', '')

      if (!endId) {
        v.onStartParams = { id: null, path: null, end: true }
        document.onmousemove = null
        document.oncontextmenu = null
        return
      }

      if (endId.split('-')[1] == '') {
        v.onStartParams = { id: null, path: null, end: true }
        document.onmousemove = null
        document.oncontextmenu = null
        return $notify({ type: 'error', text: t('notify.invalid') });
      }

      if (startId == endId) {
        v.onStartParams = { id: null, path: null, end: true }
        document.onmousemove = null
        document.oncontextmenu = null
        return $notify({ type: 'error', text: t('notify.invalid') });
      }

      let type = 1;

      let fromIpIndex = startId.split("-")[0];
      let toIpIndex = endId.split("-")[0];

      let from_ip =
        v.nodes[fromIpIndex].ip +
        ":" +
        (v.nodes[fromIpIndex].port || "80");
      let from_mac = v.nodes[fromIpIndex].mac
      let to_mac = v.nodes[toIpIndex].mac
      let to_ip =
        v.nodes[toIpIndex].ip +
        ":" +
        (v.nodes[toIpIndex].port || "80");

      let fromDiskIndex = v.nodes[fromIpIndex].disks.findIndex(e => e.id == startId.split("-")[1]);
      let toDiskIndex = v.nodes[toIpIndex].disks.findIndex(e => e.id == endId.split("-")[1]);


      // let fromDisk = v.nodes[fromIpIndex].disks[fromDiskIndex];
      // let toDisk = v.nodes[toIpIndex].disks[toDiskIndex];

      let fromPathIndex = v.nodes[fromIpIndex].disks[fromDiskIndex].paths.findIndex(e => e.id == startId.split("-")[2]);

      let toPathIndex = v.nodes[toIpIndex].disks[toDiskIndex].paths.findIndex(e => e.id == endId.split("-")[2]);


      console.log(startId.split("-"), fromPathIndex);
      let fromPath =
        v.nodes[fromIpIndex].disks[fromDiskIndex].paths[fromPathIndex].path;
      let toPath =
        v.nodes[toIpIndex].disks[toDiskIndex].paths[toPathIndex].path;

      if (fromIpIndex % 2 != 0 && toIpIndex % 2 != 0) {
        type = 2;
      } else if (fromIpIndex % 2 != 0 && toIpIndex % 2 == 0) {
        type = 3;
      }

      let position = getRelativePosition(
        "node" + fromIpIndex,
        "node" + toIpIndex
      );

      let fromcount = Math.ceil(Math.random() * 10); //随机数
      let tocount = Math.ceil(Math.random() * 10);

      if (fromcount > 7) fromcount = fromcount % 7;
      if (tocount > 7) tocount = tocount % 7;

      startId = startId + position.start + (fromcount + 1);
      endId = endId + position.end + (tocount + 1);

      v.onStartParams = { id: null, path: null, end: true }

      v.openMoveFileModal.click(); //这种开启方式会导致第一次按键被拦截

      // 创建一个名为"keydown"的键盘事件
      var event = new Event('keydown', {
        bubbles: true,
        cancelable: true,
      });

      // 设置键码为65，即按下"A"键
      Object.defineProperty(event, 'keyCode', {
        get: function () { return 65; }
      });

      // 分派事件到指定的元素，例如document.body
      document.body.dispatchEvent(event);


      let obj = Object.assign(v.moveFile, {
        from_ip,
        from_mac,
        to_mac,
        to_ip,
        sources: [fromPath],
        destination: toPath,
        startId,
        endId,
        type,
        group: props.groupId,
        validate_md5: false
      });
    }

    function handelDeletePipe(params) {
      v.tobeDelete = params;
      params.checked = true
      v.tobeDeletePipe = [params]

      // v.tobeDeletePipe = v.pipes.filter(
      //   (e) =>
      //     (e.from_ip == params.ip && e.sources[0] == params.path) ||
      //     (e.to_ip == params.ip && e.destination == params.path)
      // );
      // v.tobeDeletePipe.forEach(e => e.checked = false)
    }

    function handelUpload(params) {
      params.forEach((e) => {
        for (let key in lines) {
          const line = lines[key];
          if (line.id_from == e) {
            line.setOptions({ dash: { animation: true } });
          } else {
            line.setOptions({ dash: { animation: false } });
          }
        }
      });
    }

    function handleStart(obj) {
      v.onStartParams = obj
    }

    function linesPosition() {
      for (let key in lines) {
        let gravity = RADIAN * v.scale
        lines[key]?.position(); // 调整位置
        lines[key]?.setOptions({
          size: LISE_BASE_SIZE * v.scale,
          startSocketGravity: [lines[key]?.startSocketGravity[0] > 0 ? gravity : -gravity, 0],
          endSocketGravity: [lines[key]?.endSocketGravity[0] > 0 ? gravity : -gravity, 0],
        })
      }
    }

    function handleUpdatePosition({ index, left, top }) {
      v.nodes[index].left = left
      v.nodes[index].top = top
    }

    function handelFilterEnter() {
      if (!v.newFilter) return
      if (v.filters.includes('*')) return
      if (v.newFilter == '*') {
        v.filters = ['*']
      } else {
        v.filters.push(v.newFilter)
      }
      v.newFilter = ''
    }

    function handelAddPath(val) {
      v.tobeaddIp = val.ip;
      v.tobeaddMountpoint = val.mountpoint;
      v.tobeaddNodeId = val.node_id;
      v.tobeaddMac = val.mac;
      v.newPath = '';
    }

    watch(() => v.transform, (val) => {
      localStorage.setItem('_transform', val)
      if (v.saveing) clearTimeout(v.saveing)
      v.saveing = setTimeout(() => {
        updatePosition()
      }, 500)
    })


    // 缩放相关
    const MAX_SCALE = 1;
    const MIN_SCALE = 0.11 // 因为缩放倍率不是规则数 这样设置才能保持最小10%
    // 生成管理视图移动的对象
    function createView(params) {
      // 原本的默认属性
      let matrix = [1, 0, 0, 1, 0, 0]; // current view transform
      var scale = 1;
      var m = matrix; // alias 
      // current scale
      const pos = { x: 0, y: 0 }; // current position of origin
      var dirty = true;
      if (v.transform) {
        let old = v.transform.split(',')
        v.scale = scale = Number(old[0])
        pos.x = Number(old[4])
        pos.y = Number(old[5])
      }
      const API = {
        applyTo(el) {
          if (dirty) { this.update() }
          let result = `${m[0]},${m[1]},${m[2]},${m[3]},${m[4]},${m[5]}`
          el.style.transform = `matrix(${result})`; // 修改数字和图形变换
          v.transform = result
          linesPosition()
        },
        update() {
          dirty = false;
          m[3] = m[0] = scale;
          m[2] = m[1] = 0;
          m[4] = pos.x;
          m[5] = pos.y;
        },
        pan(amount) {
          if (dirty) { this.update() }
          pos.x += amount.x;
          pos.y += amount.y;
          dirty = true;
        },
        scaleAt(at, amount) { // at in screen coords 在屏幕内坐标
          scale *= amount;
          if (dirty) { this.update() }
          v.scale = scale
          pos.x = at.x - (at.x - pos.x) * amount;
          pos.y = at.y - (at.y - pos.y) * amount;
          dirty = true;
        },
      };
      return API;
    }

    // 使用变量方便初始化
    let view = createView()
    // 滚动缩放
    function mouseWheelEvent(event) {
      if (!props.groupId) return
      const x = event.offsetX - (zoomMe.clientWidth / 2);
      let y = event.offsetY - (zoomMe.clientHeight / 2);
      if (event.deltaY < 0) {
        if (v.scale < MAX_SCALE) {
          view.scaleAt({ x, y }, 1.1);
          view.applyTo(zoomMe);
        }
      } else {
        if (v.scale > MIN_SCALE) {
          view.scaleAt({ x, y }, 1 / 1.1);
          view.applyTo(zoomMe);
        }
      }
      event.preventDefault();
    }

    let mouseTime;
    const mouse = { x: 0, y: 0, oldX: 0, oldY: 0, button: false, };
    let distance = {};
    var origin;
    var scale = 1;
    function getDistance(start, stop) {
      return Math.sqrt(Math.pow((stop.x - start.x), 2) + Math.pow((stop.y - start.y), 2));
    }
    function getOrigin(first, second) {
      return {
        x: (first.x + second.x) / 2,
        y: (first.y + second.y) / 2
      };
    }


    function mouseEvent(event) {
      if (!props.groupId) return
      if (v.isEmpty) return
      if (event.button == 2) {
        return mouse.right = true
      }
      const nodes = document.querySelector('.nodes')
      if (event.type === "mousedown") {
        const sel = window.getSelection()
        if (sel.type == 'Range') {
          sel.removeAllRanges()
        }
        if (v.onStartParams.id) {
          handleDragEnd({})
        }
        if (mouse.right) {
          return mouse.right = false
        }
        mouse.oldX = event.pageX;
        mouse.oldY = event.pageY;
        mouse.x = event.pageX;
        mouse.y = event.pageY;
        mouse.button = true
        event.target.style.cursor = "grab";
        mouseTime = setTimeout(() => {
          event.target.style.cursor = "grabbing";
        }, 200)
        nodes.addEventListener("mousemove", mouseMove, { passive: false });
      } else if (event.type == 'touchstart') {
        if (event.targetTouches?.length > 2) return // 多手指直接无视

        if (event.targetTouches?.length == 2) {
          distance.start = getDistance({
            x: event.targetTouches[0].screenX,
            y: event.targetTouches[0].screenY
          }, {
            x: event.targetTouches[1].screenX,
            y: event.targetTouches[1].screenY
          });
        }
        if (v.onStartParams.id) {
          handleDragEnd({})
        }
        mouse.oldX = event.targetTouches[0].pageX;
        mouse.oldY = event.targetTouches[0].pageY;
        mouse.x = event.targetTouches[0].pageX;
        mouse.y = event.targetTouches[0].pageY;
        mouse.button = true
        nodes.addEventListener("touchmove", touchMove, { passive: false });
      }

      if (event.type === "mouseup" || event.type === "mouseleave" || event.type === 'touchend') {
        clearTimeout(mouseTime)
        mouse.button = false
        event.target.style.cursor = ''
        nodes.removeEventListener("mousemove", mouseMove);
        nodes.removeEventListener("touchmove", touchMove);
        linesPosition()
      }
      event.cancelable && event.preventDefault()
    }

    function mouseMove(event) {
      mouse.oldX = mouse.x;
      mouse.oldY = mouse.y;
      mouse.x = event.pageX;
      mouse.y = event.pageY;
      if (mouse.button) { // pan 
        view.pan({ x: mouse.x - mouse.oldX, y: mouse.y - mouse.oldY });
        view.applyTo(zoomMe);
      }
      event.preventDefault()
    }


    function touchMove(event) {
      if (event.targetTouches.length == 1) {
        mouse.oldX = mouse.x;
        mouse.oldY = mouse.y;
        mouse.x = event.targetTouches[0].pageX;
        mouse.y = event.targetTouches[0].pageY;

        if (mouse.button) { // pan 
          view.pan({ x: mouse.x - mouse.oldX, y: mouse.y - mouse.oldY });
          view.applyTo(zoomMe);
        }
      } else if (event.targetTouches.length == 2) {
        try {
          origin = getOrigin({
            x: event.targetTouches[0].pageX,
            y: event.targetTouches[0].pageY
          }, {
            x: event.targetTouches[1].pageX,
            y: event.targetTouches[1].pageY
          });

          distance.stop = getDistance({
            x: event.targetTouches[0].screenX,
            y: event.targetTouches[0].screenY
          }, {
            x: event.targetTouches[1].screenX,
            y: event.targetTouches[1].screenY
          });
          scale = distance.stop / distance.start;

          if (scale > 1) {
            if (v.scale < MAX_SCALE) {
              view.scaleAt({ x: origin.x, y: origin.y }, 1.1);
              view.applyTo(zoomMe);
            }
          } else {
            if (v.scale > MIN_SCALE) {
              view.scaleAt({ x: origin.x, y: origin.y }, 1 / 1.1);
              view.applyTo(zoomMe);
            }
          }
          event.preventDefault()

        } catch (error) {
          $notify({ type: 'error', text: error })
        }

      }

      event.preventDefault()
    }



    function restoreLayout() {
      const VW = document.body.clientWidth
      let width = document.querySelector('.node-f').getBoundingClientRect().width
      const PER = Math.floor(VW / width)
      v.nodes.forEach((node, index) => {
        let left, top
        if (index < PER) { top = 61 + 'px' }
        else { top = 111 + Math.floor(index / PER) * width + 'px' }
        left = (index % PER) * 600 + 21 + 'px'
        node.left = left
        node.top = top
      })
      v.scale = 1
      document.querySelector('#zoomMe').style.transform = `matrix(${DEFAULT_POSITION})`
      v.transform = DEFAULT_POSITION
      view = createView()
      localStorage.removeItem('_transform')
      setTimeout(() => createlineList(v.pipes), 10)
      confirmSave()
    }

    function backCenter() {
      let width = document.querySelector('.node-f').getBoundingClientRect().width
      let height = document.querySelector('.node-f').getBoundingClientRect().height

      let minTop = v.nodes.reduce((prev, current) => (Math.ceil(prev.top.replace('px', '')) < Math.ceil(current.top.replace('px', ''))) ? prev : current);
      let maxTop = v.nodes.reduce((prev, current) => (Math.ceil(prev.top.replace('px', '')) > Math.ceil(current.top.replace('px', ''))) ? prev : current);
      let minLeft = v.nodes.reduce((prev, current) => (Math.ceil(prev.left.replace('px', '')) < Math.ceil(current.left.replace('px', ''))) ? prev : current);
      let maxLeft = v.nodes.reduce((prev, current) => (Math.ceil(prev.left.replace('px', '')) > Math.ceil(current.left.replace('px', ''))) ? prev : current);
      let list = v.transform?.split(',') || ['1', '0', '0', '1', '0', '0']
      let scale = list[0]
      // 画布左上角的位置
      let zoomMeX = document.querySelector('#zoomMe').getBoundingClientRect().left
      let zoomMeY = document.querySelector('#zoomMe').getBoundingClientRect().top - 111  // 不减111就是屏幕中心 减掉后是基于nodes的中心

      // console.log('zoomMeX', zoomMeX, zoomMeY);
      // 先找到中心点 
      let left = Number(minLeft.left.replace('px', '')) * scale + zoomMeX + Number(maxLeft.left.replace('px', '')) * scale + zoomMeX + width
      let top = Number(minTop.top.replace('px', '')) * scale + zoomMeY + Number(maxTop.top.replace('px', '')) * scale + zoomMeY + height

      left = left / 2
      top = top / 2  // 这里就是画布中心的x,y
      // console.log(left, top);

      // 寻找视口的中心 
      const nodes = document.querySelector('.nodes')
      let centerX = nodes.clientWidth / 2  // 862.5
      let centerY = nodes.clientHeight / 2 // 530.5

      let x = left - centerX
      let y = top - centerY

      console.log('off', x, y);

      list[4] = list[4] - x
      list[5] = list[5] - y
      // console.log(list[4], list[5]);
      v.transform = list.join(',')
      localStorage.setItem('_transform', v.transform)
      document.querySelector('#zoomMe').style.transform = `matrix(${v.transform})`
      view = createView()
      setTimeout(linesPosition, 10)
      updatePosition()
    }

    function handelMoveFileClose() {
      v.newFilter = '';
      v.filters = [];
      v.onStartParams = { id: null, path: null }
      v.moveFile = { force_over_write: false, filters_type: 'exclude', sync_type: 'relative' };
    }


    const theme = computed(() => store.state.theme)

    function handleDblClick(e) {
      if (e.detail >= 2) e.preventDefault()
    }

    function handleFilterInput() {
      if (v.filters.includes('*')) {
        v.moveFile.filterError = 'The filters already contain *, entering another will conflict'
      } else {
        v.moveFile.filterError = null
      }
      if (!v.newFilter) {
        v.moveFile.filterError = null
      }
    }


    onMounted(() => {
      document.querySelector('#zoomMe').style.transform = `matrix(${v.transform})`
      const nodes = document.querySelector('.nodes')
      document.body.classList.add("overflow-hidden")
      // nodes.addEventListener("mousemove", mouseEvent, { passive: false });
      //nodes.addEventListener("mousedown", mouseEvent, { passive: false });
      nodes.addEventListener("mouseup", mouseEvent, { passive: false });
      nodes.addEventListener("touchend", mouseEvent, { passive: false });
      nodes.addEventListener("mouseleave", mouseEvent, { passive: false });
      nodes.addEventListener("wheel", mouseWheelEvent, { passive: false });
    })

    onActivated(() => {
      console.log('activated');
      // document.body.style.overflow = 'hidden'
    })


    onUnmounted(() => {
      for (let key in lines) {
        lines[key]?.remove(); // 清理旧线
      }
      v.linesList = []
      lines = {}
      document.body.classList.remove("overflow-hidden")
    })

    return {
      ...toRefs(v),
      nodedata,
      handelenter,
      handleleave,
      handelAddPath,
      deleteNode,
      handelDeletePipe,
      createPath,
      deletePath,
      deleteDisk,
      handelMove,
      handleCreateLine,
      handleDragEnd,
      handelMoveFile,
      handelMoveFileClose,
      handelUpload,
      deletePipe,
      reloadline,
      handleStart,
      getNodes, putName, handleUpdatePosition,
      updateNode,
      handelFilterEnter, restoreLayout, confirmSave, backCenter,
      mouseEvent, theme, handleDblClick, handleFilterInput, handelCloseDeletePipeModal
      , handleScrollReloadLine, t
    };
  },
};
</script>

<template>
  <div class="nodes" @mousedown.self="mouseEvent" @touchstart.self="mouseEvent">
    <Empty v-if="isEmpty" class="position-absolute top-50 start-50 translate-middle" />
    <div class id="zoomMe" @mousedown.self="mouseEvent">
      <Node v-for="(item, index) in nodes" :key="index" :data="item" :poddList="poddList" :id="index" :group="groupId"
        :scale="scale" :onStartParams="onStartParams" @delete="(val) => tobeDelete = val" @enter="handelenter"
        @leave="handleleave" @addpath="handelAddPath" @deletePath="(val) => (tobeDelete = val)"
        @putname="(val) => { oldName = val.oldName; newName = val.oldName; tobeaddIp = val.ip; tobeaddNodeId = val.node_id; }"
        @onMove="handelMove" @onMoveEnd="handelMove('end')" @createLine="handleCreateLine" @dragEnd="handleDragEnd"
        @scroll="handleScrollReloadLine" @onStart="handleStart" @updatePosition="handleUpdatePosition" />
    </div>
  </div>
  <div class="position-absolute bottom-0 end-0 m-5" v-if="!isEmpty && groupId">
    <Loading v-if="loadnodesing" />
    <div class="dropdown" v-else>
      <button class="badge border rounded-circle p-2 py-3 d-flex align-items-center justify-content-center floating-menu"
        type="button" data-bs-toggle="dropdown" style="width: 40px; height: 40px;">
        <span class="" v-if="scaleChange">{{ parseInt(scale * 100) }}%</span>
        <i class="bi bi-three-dots fs-5" v-else></i>
      </button>
      <ul class="dropdown-menu" :class="[theme == 'dark' && 'dropdown-menu-dark',]" aria-labelledby="dropdownMenuButton2">
        <li class="dropdown-item" @click.self="backCenter">
          <i class="bi bi-align-center"></i> {{ t('nodes.backC') }}
        </li>
        <li class="dropdown-item" data-bs-toggle="modal" data-bs-target="#confirmSaveModal">
          <i class="bi bi-repeat"></i> {{ t('nodes.resetL') }}
        </li>
      </ul>
    </div>
  </div>
  <div @mousedown.stop="handleDblClick">
    <!-- Modal -->
    <div class="modal fade" id="putNameModal" tabindex="-1">
      <div class="modal-dialog modal-dialog-centered">
        <div class="modal-content">
          <div class="modal-header">
            <h5 class="modal-title">{{ t('nodes.nameModal.title') }}</h5>
            <button type="button" class="btn-close" data-bs-dismiss="modal" ref="closeName"
              @click="newName = null;"></button>
          </div>
          <div class="modal-body">
            <div class="mb-3">
              <label class="form-label">{{ t('nodes.nameModal.label') }} {{ tobeaddIp }}</label>
              <input type="text" class="form-control" :class="{ 'is-invalid': newPathErr }" maxlength="128"
                v-model.trim="newName" @keydown.enter="putName" />
            </div>
          </div>
          <div class="modal-footer">
            <button type="button" class="btn border" data-bs-dismiss="modal" @click="newName = null; ">
              {{ t('cancel') }}
            </button>
            <button type="button" class="btn btn-primary" @click="putName"
              :disabled="!newName || newName == oldName || loading">
              <Loading :sm="true" v-if="loading" />
              {{ t('save') }}
            </button>
          </div>
        </div>
      </div>
    </div>
    <!-- Modal -->
    <div class="modal fade" id="deletePathModal" tabindex="-1" data-bs-backdrop="static" data-bs-keyboard="true">
      <div class="modal-dialog modal-dialog-centered">
        <div class="modal-content">
          <div class="modal-header">
            <h5 class="modal-title">{{ t('nodes.deletePModal.title') }}</h5>
            <button type="button" class="btn-close" data-bs-dismiss="modal" ref="closeDeletePath"
              @click="deleteSource = false"></button>
          </div>

          <div class="modal-body">
            <div class="alert alert-danger" role="alert">
              <h4 class="alert-heading">
                <i class="bi bi-exclamation-triangle-fill"></i>
                {{ t('nodes.deletePModal.tip') }}
              </h4>
              <p>
                <strong class="text-uppercase"> {{ t('console.path') }} </strong>: <span class="text-break"> {{
                  tobeDelete.path }} </span> <br />
                <strong> {{ t('nodes.deletePModal.fromMou') }}</strong>: {{ tobeDelete.mountpoint
                }}<br />
                <strong>{{ t('nodes.deletePModal.fromIp') }}</strong>: {{ tobeDelete.ip }}
              </p>
              <div class="form-check">
                <input class="form-check-input" type="checkbox" v-model="deleteSource" id="flexCheckDefault" />
                <label class="form-check-label" for="flexCheckDefault">
                  {{ t('nodes.deletePModal.delSource') }}
                </label>
              </div>
              <hr />
              <p class="mb-0">{{ t('undone') }}</p>
            </div>
          </div>
          <div class="modal-footer">
            <button type="button" class="btn border" data-bs-dismiss="modal" @click="deleteSource = false">
              {{ t('cancel') }}
            </button>
            <button type="button" class="btn btn-danger" @click="deletePath" :disabled="loading">
              <Loading :sm="true" v-if="loading" />
              {{ t('sure') }}
            </button>
          </div>
        </div>
      </div>
    </div>
    <!-- Modal -->
    <div class="modal fade" id="createPathModal" tabindex="-1" data-bs-backdrop="static" data-bs-keyboard="true">
      <div class="modal-dialog modal-dialog-centered">
        <div class="modal-content ">
          <div class="modal-header">
            <h5 class="modal-title">{{ t('nodes.cPathModal.title') }}</h5>
            <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Close" ref="closePath"
              @click="newPath = null; "></button>
          </div>
          <div class="modal-body">
            <div class="mb-3">
              <label class="form-label">{{ t('nodes.cPathModal.label') }} {{ tobeaddIp }}{{ tobeaddMountpoint }}</label>
              <input type="text" class="form-control" :class="{ 'is-invalid': newPathErr }" v-model.trim="newPath"
                @keydown.enter="createPath" />
            </div>
          </div>
          <div class="modal-footer">
            <button type="button" class="btn border" data-bs-dismiss="modal" @click="newPath = null">
              {{ t('cancel') }}
            </button>
            <button type="button" class="btn btn-primary" :disabled="!newPath || loading" @click="createPath">
              <Loading :sm="true" v-if="loading" />
              {{ t('nodes.cPathModal.cPath') }}
            </button>
          </div>
        </div>
      </div>
    </div>
    <!-- Modal -->
    <div class="modal fade" id="deleteNodeModal" tabindex="-1" data-bs-backdrop="static" data-bs-keyboard="true">
      <div class="modal-dialog modal-dialog-centered">
        <div class="modal-content">
          <div class="modal-header">
            <h5 class="modal-title">{{ t('nodes.delNodeModal.title') }}</h5>
            <button type="button" class="btn-close" data-bs-dismiss="modal" ref="closeDelete"></button>
          </div>

          <div class="modal-body">
            <div class="alert alert-danger" role="alert">
              <h4 class="alert-heading">
                <i class="bi bi-exclamation-triangle-fill"></i> {{ t('nodes.delNodeModal.tip') }}
              </h4>
              <div class="text-wrap">
                <div>
                  {{ t('nodes.IP') }}: {{ tobeDelete.ip }}
                </div>
                <div style="word-wrap: break-word;">{{ t('nodes.name') }}: {{ tobeDelete.name }}
                </div>
                <div>
                  {{ t('nodes.mac') }}: {{ tobeDelete.mac }}
                </div>
              </div>
              <hr />
              <p class="mb-0">{{ t('nodes.delNodeModal.text') }}</p>
            </div>
          </div>
          <div class="modal-footer">
            <button type="button" class="btn border" data-bs-dismiss="modal">
              {{ t('cancel') }}
            </button>
            <button type="button" class="btn btn-danger" @click="deleteNode" :disabled="loading">
              <Loading :sm="true" v-if="loading" />
              {{ t('sure') }}
            </button>
          </div>
        </div>
      </div>
    </div>
    <!-- Modal -->
    <div class="modal fade" id="deleteDiskModal" tabindex="-1" data-bs-backdrop="static" data-bs-keyboard="true">
      <div class="modal-dialog modal-dialog-centered">
        <div class="modal-content">
          <div class="modal-header">
            <h5 class="modal-title">{{ t('nodes.delDiskModal.title') }}</h5>
            <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Close"
              ref="closeDeleteDisk"></button>
          </div>

          <div class="modal-body">
            <div class="alert alert-danger" role="alert">
              <h4 class="alert-heading">
                <i class="bi bi-exclamation-triangle-fill"></i>
                {{ t('nodes.delDiskModal.tip') }} <strong>{{ t('nodes.delDiskModal.strong') }}</strong>?
              </h4>
              <p>
                <strong> {{ t('nodes.mountP') }} </strong>: {{ tobeDelete.mountpointName }}<br />
                <strong>{{ t('nodes.fromIp') }} </strong>: {{ tobeDelete.ip }}
              </p>
              <hr />
              <p class="mb-0">{{ t('undone') }}</p>
            </div>
          </div>
          <div class="modal-footer">
            <button type="button" class="btn border" data-bs-dismiss="modal">
              {{ t('cancel') }}
            </button>
            <button type="button" class="btn btn-danger" @click="deleteDisk" :disabled="loading">
              <Loading :sm="true" v-if="loading" />
              {{ t('sure') }}
            </button>
          </div>
        </div>
      </div>
    </div>
    <!-- reset layout -->
    <div class="modal fade" id="confirmSaveModal" tabindex="-1" data-bs-backdrop="static" data-bs-keyboard="true">
      <div class="modal-dialog modal-dialog-centered">
        <div class="modal-content">
          <div class="modal-header">
            <h5 class="modal-title">{{ t('nodes.resetLModal.title') }}</h5>
            <button type="button" class="btn-close" data-bs-dismiss="modal" ref="closeConfimSave"></button>
          </div>
          <div class="modal-body">
            <div class="alert alert-secondary mb-0" role="alert">
              <div class="alert-heading">
                <i class="bi bi-question-circle-fill"></i>
                {{ t('nodes.resetLModal.tip') }}
              </div>
            </div>
          </div>
          <div class="modal-footer">
            <button type="button" class="btn border" data-bs-dismiss="modal">
              {{ t('cancel') }}
            </button>
            <button type="button" class="btn btn-primary" @click="restoreLayout" :disabled="loading">
              <Loading :sm="true" v-if="loading" />
              {{ t('sure') }}
            </button>
          </div>
        </div>
      </div>
    </div>
    <!-- Modal -->
    <div class="modal fade" id="MoveFileModal" tabindex="-1" data-bs-backdrop="static" data-bs-keyboard="false">
      <div class="modal-dialog modal-dialog-centered">
        <div class="modal-content">
          <div class="modal-header">
            <h5 class="modal-title">{{ t('nodes.mfModal.title') }}</h5>
            <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Close" ref="closeMoveFileModal"
              @click="handelMoveFileClose"></button>
          </div>
          <div class="modal-body">
            <h4>{{ t('nodes.mfModal.title1') }}</h4>
            <div class="text-break">
              <strong>{{ t('nodes.from') }}:</strong> {{ moveFile.from_ip }}:{{ moveFile.sources && moveFile.sources[0] }}

            </div>
            <div class=" text-break">
              <strong> {{ t('nodes.to') }}:</strong> {{ moveFile.to_ip }}:{{ moveFile.destination }}
            </div>

            <div class="my-3">
              <h4>{{ t('nodes.mfModal.title2') }}</h4>
              <div class="form-check">
                <input class="form-check-input border" type="radio" v-model="moveFile.sync_type" name="flexRadioDefault"
                  id="relativeSync" value="relative" />
                <label class="form-check-label" for="relativeSync">
                  {{ t('nodes.mfModal.label1') }}
                  <span class="form-text">{{ t('nodes.mfModal.formText1') }}</span>
                </label>
              </div>
              <div class="form-check">
                <input class="form-check-input border" type="radio" v-model="moveFile.sync_type" name="flexRadioDefault"
                  id="absoluteSync" value="absolute" />
                <label class="form-check-label" for="absoluteSync">
                  {{ t('nodes.mfModal.label2') }}
                  <span class="form-text"> {{ t('nodes.mfModal.formText2') }}</span>
                </label>
              </div>
            </div>

            <div class="my-3">
              <h4>{{ t('nodes.mfModal.title3') }}</h4>
              <div class="form-check ">
                <input class="form-check-input border" type="checkbox" id="forceOverwrite"
                  v-model="moveFile.force_over_write" />
                <label class="form-check-label" for="forceOverwrite">
                  {{ t('nodes.mfModal.label3') }}
                </label>
              </div>
            </div>

            <div class="my-3">
              <h4>{{ t('nodes.mfModal.title4') }}</h4>
              <div class="form-check ">
                <input class="form-check-input border" type="checkbox" id="md5" v-model="moveFile.validate_md5" />
                <label class="form-check-label" for="md5">
                  {{ t('nodes.mfModal.label4') }}
                </label>
              </div>
            </div>
            <div class="my-3">
              <h4>{{ t('nodes.mfModal.title5') }}</h4>
              <div class="form-text mt-0 mb-2">
                {{ t('nodes.mfModal.formText5') }}
              </div>
              <div class="form-check form-check-inline ">
                <input class="form-check-input border" type="radio" name="flexRadioDefault2"
                  v-model="moveFile.filters_type" value="exclude" id="exclude" @change="filters = []; newFilter = '' ">
                <label class="form-check-label" for="exclude">
                  {{ t('nodes.mfModal.label5') }}
                </label>
              </div>
              <div class="form-check form-check-inline">
                <input class="form-check-input border" type="radio" v-model="moveFile.filters_type"
                  name="flexRadioDefault2" value="include" id="include" @change="filters = []; newFilter = '' ">
                <label class="form-check-label" for="include">
                  {{ t('nodes.mfModal.label6') }}
                </label>
              </div>
              <label class="d-flex align-items-center mb-2">{{ t('nodes.mfModal.label7') }}
                <tippy :allowHTML="true" theme='dark' :maxWidth="800" arrow>
                  <template #content>
                    <WildcardTable />
                  </template>
                  <i class="bi bi-info-circle mx-2"></i>
                </tippy>
              </label>
              <div class="input-group position-relative mb-2">
                <input type="text" class="form-control" :class="{ 'is-invalid': moveFile.filterError }"
                  :placeholder="t('nodes.mfModal.placeHolder')" @keydown.enter="handelFilterEnter"
                  @input="handleFilterInput" v-model.trim="newFilter">
                <div class="invalid-tooltip rounded">
                  {{ moveFile.filterError }}
                </div>
                <button class="btn btn-outline-secondary" type="button" id="button-addon2"
                  :disabled="!newFilter || moveFile.filterError" @click="filters.push(newFilter); newFilter = '' ">
                  <i class="bi bi-plus-lg"></i>
                </button>
              </div>
              <div class="d-flex align-items-center flex-wrap" style="min-height: 40px" v-if="filters.length > 0">
                <div class=" position-relative border border-r2 me-3 mb-2" v-for="( item, index ) in  filters "
                  :key="index" style="height:28px;max-width: 100%;">
                  <div class="px-3 py-1 position-relative d-flex align-items-center">
                    <div class="w-100 text-truncate "> {{ item }}</div>
                    <span
                      class="cursor-pointer position-absolute top-0 start-100 translate-middle badge rounded-pill bg-danger fs-sm"
                      @click="filters.splice(index, 1)">x</span>
                  </div>

                </div>
              </div>

              <div class="alert alert-danger p-2" role="alert"
                v-if="filters.length == 0 && moveFile.filters_type == 'include'">
                <i class="bi bi-exclamation-diamond-fill"></i> {{ t('nodes.mfModal.alertDanger') }}
              </div>
            </div>

          </div>
          <div class=" modal-footer">
            <button type="button" class="btn border" data-bs-dismiss="modal" @click="handelMoveFileClose">
              {{ t('cancel') }}
            </button>
            <button type="button" class="btn btn-primary" data-bs-dismiss="modal" @click="handelMoveFile"
              :disabled="loading || filters.length == 0 && moveFile.filters_type == 'include'">
              <Loading :sm="true" v-if="loading" />
              {{ t('submit') }}
            </button>
          </div>
        </div>
      </div>
    </div>

    <!-- Modal -->
    <div class="modal fade" id="UpdatePipeModal" tabindex="-1" data-bs-keyboard="true">
      <div class="modal-dialog modal-dialog-centered">
        <PipeInfo :pipe="tobeUpdatePipe" :readonly="true" v-if="tobeUpdatePipe.id" @deletePipe="handelDeletePipe" />
      </div>
    </div>
    <!-- deletepipeModal -->
    <div class="modal fade" id="deletePipeModal" tabindex="-1" data-bs-backdrop="static" data-bs-keyboard="false">
      <div class="modal-dialog modal-dialog-centered">
        <div class="modal-content">
          <div class="modal-header">
            <h5 class="modal-title">{{ t('nodes.delPipeModal.title') }}</h5>
            <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Close" ref="closeDeltePipeModal"
              @click="handelCloseDeletePipeModal"></button>
          </div>
          <div class="modal-body">
            <div class="alert alert-danger" role="alert">
              <h4 class="alert-heading">
                <i class="bi bi-exclamation-triangle-fill"></i>
                {{ t('nodes.delPipeModal.tip') }}
              </h4>
              <p></p>
              <!-- <strong class="w-100 text-break mb-1">Pipe related to: {{ tobeDelete.ip}}{{ tobeDelete.path?.startsWith('/') ? '' : tobeDelete.startStr }}{{ tobeDelete.path }}:</strong> -->
              <div class="" v-for="( item, index ) in  tobeDeletePipe " :key="index">
                <label class="form-check-label" :for="item.id">
                  <strong>{{ t('nodes.from') }}:</strong> {{ item.from_ip }}:{{ item.sources[0].startsWith('/') ? '' :
                    tobeDelete.startStr
                  }}{{ item.sources[0] }}
                  <br />
                  <strong>{{ t('nodes.to') }}:</strong> {{ item.to_ip }}:{{ tobeDelete.path?.startsWith('/') ? '' :
                    tobeDelete.startStr
                  }}{{ item.destination }}
                </label>
              </div>
              <div v-if="tobeDeletePipe.length == 0" class="text-center my-4">
                No records can be deleted
              </div>
              <hr />
              <p class="mb-0">{{ t('undone') }}</p>
            </div>
          </div>
          <div class="modal-footer">
            <button type="button" class="btn border" @click="handelCloseDeletePipeModal" data-bs-toggle="modal"
              data-bs-target="#UpdatePipeModal">
              {{ t('cancel') }}
            </button>
            <button type="button" class="btn btn-danger" @click="deletePipe" :disabled="loading">
              <Loading :sm="true" v-if="loading" /> {{ t('sure') }}
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>

  <!-- movefile -->
  <button data-bs-toggle="modal" ref="openMoveFileModal" data-bs-target="#MoveFileModal" hidden></button>
  <!-- pipeinfo -->
  <button ref="openUpdateFileModal" data-bs-toggle="modal" data-bs-target="#UpdatePipeModal" hidden></button>
  <div class="offcanvas offcanvas-end progressing" tabindex="-1" id="offcanvasRight" data-bs-backdrop="false">
    <div class="offcanvas-header">
      <h5 id="offcanvasRightLabel">Progressing</h5>
      <i class="bi bi-chevron-right cursor-pointer" style="font-size: 20px" data-bs-dismiss="offcanvas"></i>
    </div>
    <div class="offcanvas-body">
      <!-- <UploadList @upload="handelUpload" :groupId="groupId" v-if="groupId" /> -->
    </div>
  </div>
  <div id="to_top"></div>
</template>
<style lang="scss" scoped>
.nodes {
  position: relative;
  height: calc(100vh - 50px);
  overflow: hidden;
}

#zoomMe {
  position: absolute;
  left: 0;
  top: 0;
  width: 100px;
  height: 100px;
  z-index: 1;
  // background-color: gray;
}

.progressing {
  background-color: var(--progressing-bg);
  width: 353px;
  margin-top: 60px;
  font-size: 12px;
}

.floating-menu {
  color: var(--color);
  background-color: var(--menu-bg);
  border-color: var(--color);
}
</style>