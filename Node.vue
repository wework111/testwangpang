111
<template>
  <div class="node-f  position-absolute" :id="'node' + id" :style="{ left, top, 'z-index': z }">
    <div class="node border-r2" @mousedown="handleNodeAlive" @touchstart="handleNodeAlive">
      <div class="d-flex justify-content-between align-items-center py-2 px-2 cursor-grab" @mousedown="handeleMove"
        @touchstart="handeleMove">
        <div class="d-flex align-items-center  user-select-none" style="width:80%">
          <div class="me-2">
            <i class="bi bi-windows" v-if="data.system == 'windows'"></i>
            <i class="bi bi-linux d-block" style="width:1rem;height: 1rem;" v-if="data.system == 'linux'"></i>
            <i class="bi bi-apple" v-if="data.system == 'darwin'"></i>
          </div>
          <div class="d-flex   align-items-center me-2 text-truncate" style="max-width: 33.3%">
            <tippy :content="'IP: ' + data.ip + ':' + data.port" :theme="theme" sticky>
              {{ data.ip }}:{{ data.port }}
            </tippy>
          </div>
          <div class="d-flex align-items-center node-info me-2 " style="max-width: 33.3%">
            <tippy :content="t('nodes.name') + ': ' + data.name" :theme="theme" sticky class="text-truncate">
              {{ data.name }}
            </tippy>
          </div>
          <div class="d-flex  align-items-center me-2 text-truncate" style="max-width: 25%">
            <tippy :content="t('node.version') + ': ' + data.version" :theme="theme" sticky class="text-truncate">
              {{ data.version }}
            </tippy>
          </div>
          <div class="d-flex  align-items-center me-2 text-truncate" style="max-width: 25%">
            <tippy :content="t('node.state') + ': ' + data.state" :theme="theme" sticky class="text-truncate">
              {{ data.state }}
            </tippy>
          </div>
        </div>
        <div class="d-flex align-items-center " @mousedown.stop @touchstart.stop
          :class="{ 'disable': onStartParams.id || scale < 0.5 }">
          <span class="text-success me-1" v-if="saving == 'saved'">{{ t('saved') }}</span>
          <Loading class="me-2" :sm="true" v-else-if="saving" />
          <tippy :allowHTML="true" trigger="click" touch="hold" :theme="theme + 'menu'" placement="left" interactive
            :zIndex="99" class="lh-1" sticky :appendTo="appendTo" :offset="[0, 0]" :arrow="false"
            :onShow="onPathMenuShow">
            <i class="bi-gear-fill cursor-pointer"></i>
            <template #content="{ hide }">
              <div>
                <div class="px-3 py-1 dropdown-item"
                  @click="$emit('addpath', { ip: data.ip, node_id: data.id, mac: data.mac, }); hide()"
                  data-bs-toggle="modal" data-bs-target="#createPathModal">
                  <i class="bi bi-plus-lg"></i>
                  {{ t('node.newPath') }}
                </div>
                <div class="px-3 py-1 dropdown-item" data-bs-toggle="modal" data-bs-target="#putNameModal"
                  @click="$emit('putname', { ip: data.ip, node_id: data.id, oldName: data.name }); hide(); ">
                  <i class="bi bi-pencil-square"></i>
                  {{ t('node.setNodeName') }}
                </div>
                <div class="px-3 py-1 dropdown-item" @click="$emit('delete', data); hide() " data-bs-toggle="modal"
                  data-bs-target="#deleteNodeModal" v-if="data.state == 'Offline'">
                  <i class="bi bi-x-lg"></i>
                  {{ t('node.deleteNode') }}
                </div>
              </div>
            </template>
          </tippy>
        </div>
      </div>
      <div class="position-absolute line-station" style="left:2rem" :id="'node-left-t-s' + id" />
      <div class="position-absolute line-station" style="left:2rem" :id="'node-left-t-e' + id" />
      <div class="position-absolute line-station" style="right:2rem;" :id="'node-right-t-s' + id" />
      <div class="position-absolute line-station" style="right:2rem;" :id="'node-right-t-e' + id" />
      <div class="position-absolute line-station" style="left:2rem;bottom: 0;" :id="'node-left-b-s' + id" />
      <div class="position-absolute line-station" style="left:2rem;bottom: 0;" :id="'node-left-b-e' + id" />
      <div class="position-absolute line-station" style="right:2rem;bottom: 0;" :id="'node-right-b-s' + id" />
      <div class="position-absolute line-station" style="right:2rem;bottom: 0;" :id="'node-right-b-e' + id" />
      <div class="disk-f position-relative" @scroll.stop="handleScroll" @mousewheel.stop="null" @wheel.stop="null">
        <!-- <div class="my-3 text-center position-absolute top-50 start-50 translate-middle"
          v-if="data.disks.length == 0 || !data.disks">
          <Empty />
        </div> -->
        <div v-for="(disk, i) in data.disks" :key="i" v-show="disk.paths.length > 0">
          <div class="border-r2 disk p-2 px-2 mx-2 mt-1">
            <div class="w-100 diskSpace" ref="diskSpace" style="height:4px" v-if="disk.id > 0">
              <div class="disk-usage" ref="diskUsage" :style="{ width: (disk.space_percentage * 100) + '%' }"
                :class="[{ 'bg-primary': disk.space_percentage <= 0.8 }, { 'bg-red': disk.space_percentage > 0.8 }]">
              </div>
            </div>
            <div class="d-flex justify-content-between">
              <div :class="{ 'user-select-none': onStartParams.id || $store.state.isDraging }">
                <i class="bi bi-device-hdd-fill" />
                {{ disk.mountpoint }}

                <tippy :content="t('node.tempatureTip', { d: disk.tempature })" v-show="disk.tempature > 50">
                  <i class="bi bi-info-circle-fill text-danger"></i>
                </tippy>

              </div>
              <div class="d-flex align-items-center" :class="{ 'disable': onStartParams.id || scale < 0.5 }">
                <tippy :appendTo="appendTo" :allowHTML="true" trigger="click" :theme="theme + 'menu'" placement="left"
                  touch="hold" interactive :arrow="false" :zIndex="99" :offset="[0, 0]" sticky class="lh-1"
                  :onShow="onPathMenuShow">
                  <i class="bi-gear-fill  cursor-pointer "></i>
                  <template #content>
                    <div>
                      <div class="px-3 py-1 dropdown-item" data-bs-toggle="modal" data-bs-target="#deleteDiskModal"
                        @click="$emit('deletePath', { ip: data.ip, node_id: data.id, mac: data.mac, mountpoint: disk.id, mountpointName: disk.mountpoint })">
                        <i class="bi bi-x-lg"></i>
                        {{ t('node.deleteAll') }}
                      </div>
                    </div>
                  </template>
                </tippy>
              </div>
            </div>
            <div class="position-relative path-f" v-for="(item, index) in  disk.paths" :key="index"
              @mouseenter="$emit('enter', id + '-' + disk.id + '-' + item.id)"
              @mouseleave="$emit('leave', id + '-' + disk.id + '-' + item.id)">
              <div class="position-absolute h-100 start-100 d-flex align-items-center flex-column  ">
                <div class="mb-2" style="width: 1px; height: 1px" v-for="d in 7" :key="d"
                  :id="id + '-' + disk.id + '-' + item.id + 'right' + d" />
              </div>
              <div class="position-absolute h-100 start-0 d-flex  align-items-center flex-column  ">
                <div class="mb-1" style="width: 1px; height: 1px" v-for=" d  in  7 " :key="d"
                  :id="id + '-' + disk.id + '-' + item.id + 'left' + d"></div>
              </div>

              <div class="path border-r2 py-1 px-1 mt-2 d-flex justify-content-between " :data-path-id="'path' + item.id"
                :id="id + 'path' + disk.id + '-' + item.id"
                @mouseup="onEnd($event, id + '-' + disk.id + '-' + item.id, item.path)"
                :class="{ 'opacity-50': item.opt == 'r-' && onStartParams.id, 'border-heightlight': item.heightLight }">
                <div class="d-flex  justify-content-between align-items-center w-100 px-2">
                  <div class="">
                    <!-- Ok but msg -->
                    <tippy :content="item.description" v-if="item.state == 'Ok' && item.description" :theme="theme">
                      <div class="state-box work-warning" />
                    </tippy>
                    <!-- Working -->
                    <tippy content="Working" v-if="item.state == 'Ok' && !item.description" :theme="theme">
                      <div class="state-box working"></div>
                    </tippy>
                    <!-- Error -->
                    <tippy :content="item.description" v-if="item.state == 'Error'" :theme="theme">
                      <div class="state-box error"></div>
                    </tippy>
                    <!-- Offline  -->
                    <tippy content="Disk offline" v-if="item.state == 'Offline'" :theme="theme">
                      <div class="state-box offline"></div>
                    </tippy>
                    <!-- Waiting -->
                    <tippy content="Waiting" v-if="item.state == 'Waiting'" :theme="theme">
                      <div class="spinner-grow spinner-grow-sm" role="status">
                        <span class="visually-hidden">Loading...</span>
                      </div>
                    </tippy>
                  </div>
                  <div class="w-85 text-break d-flex align-items-center justify-content-between"
                    :class="{ 'opacity-50': item.state != 'Ok' || !disk.mountpoint, 'user-select-none': onStartParams.id || $store.state.isDraging }">
                    <span> {{ item.path }} </span>
                    <tippy content="This disk can only be read but not written" :theme="theme" v-if="item.opt == 'r-'">
                      <span class="badge bg-warning text-dark ms-2">Read only</span>
                    </tippy>
                    <span class="opacity-50 d-flex align-items-center" style="font-size: 0.25rem">{{ item.size
                    }}</span>

                  </div>

                  <tippy :allowHTML="true" trigger="click" touch="hold" :theme="theme + 'menu'" placement="left"
                    interactive class="lh-1" :zIndex="99" :appendTo="appendTo" :offset="[0, 0]" :arrow="false" sticky
                    :class="{ 'disable': onStartParams.id || scale < 0.5 }" :onShow="onPathMenuShow"
                    :onHide="onPathMenuHide">
                    <i class="bi-gear-fill cursor-pointer"></i>
                    <template #content="{ hide }">
                      <div>
                        <div class="px-3 py-1 dropdown-item" data-bs-toggle="modal" data-bs-target="#deletePathModal"
                          @click="
                            $emit('deletePath', {
                              ip: data.ip,
                              mountpoint: disk.mountpoint,
                              node_id: data.id,
                              path_id: item.id,
                              path: item.path
                            }); hide()">
                          <i class="bi bi-x-lg"></i>
                          {{ t('node.delPath') }}
                        </div>
                      </div>
                    </template>
                  </tippy>
                </div>
              </div>
              <div :id="id + '-' + disk.id + '-' + item.id" v-show="!onStartParams.id && item.opt != 'r-'"
                class="podd position-absolute top-50 start-100"
                @mousedown.stop="onStart($event, { id: id + '-' + disk.id + '-' + item.id + 'shadow', path: item.path })"
                @touchstart.stop="onStart($event, { id: id + '-' + disk.id + '-' + item.id + 'shadow', path: item.path })"
                :class="{ disable: scale < 0.5 }" v-if="item.state == 'Ok' && !item.description"></div>
              <div :id="id + '-' + disk.id + '-' + item.id + 'shadow'"
                class="position-absolute podd-shadow top-50 start-100"
                :class="{ 'lineing': onStartParams.id == `${id}-${disk.id}-${item.id}shadow` }"
                v-if="item.state == 'Ok' && !item.description">
              </div>
            </div>

          </div>
          <div class="mt-3"></div>
        </div>

      </div>
    </div>
  </div>
</template>
<script>
import { reactive, toRefs, inject, onMounted, watch, nextTick, computed, onBeforeMount } from "vue";
import LeaderLine from "leader-line-vue";
import { useStore } from "vuex";
import { getZoomCenterPoint, isScrollShow, } from "../utils/utils";
import { useTippy } from "vue-tippy";
import { left } from "@popperjs/core";
import { useI18n } from "vue-i18n";
export default {
  props: {
    id: Number, // 这里的id不是node的id 只是排序的index
    scale: Number,
    data: Object,
    poddList: Array,
    group: Number,
    onStartParams: Object,
  },
  emits: [
    "enter",
    "leave",
    "delete",
    "addpath",
    "deletePath",
    "deletePipe",
    "onMove",
    "createLine",
    "dragEnd",
    'scroll',
    'onStart', 'updatePosition', 'putname', 'onMoveEnd',
  ],
  setup(props, { emit }) {
    const { $axios, $notify } = inject("$");
    const { t, locale } = useI18n()
    const store = useStore()
    const v = reactive({
      list: [],
      editName: false,
      newName: "",
      loading: false,
      drap: false,
      onStartId: null,
      saving: false,
      left: null,
      top: null,
      z: props.data.zIndex,
      diskSpace: [],
      diskUsage: [],
      onShowTippy: {},
    });

    function handlenter() {
      emit("enter", "line1");
    }

    function handlout() {
      emit("leave", "line1");
    }

    function handleScroll(e) {
      // e.stopPropagation()
      if (v.onShowTippy.hide) {
        v.onShowTippy.hide()
      }
      if (v.onStartId) {
        for (let key in lines) {
          lines[key]?.remove(); // 清理旧线
          delete lines[key]
        }
        let position = isScrollShow(props.onStartParams.id)
        let startId = v.onStartId
        if (!position.show) {
          delete lines[v.onStartId + '~to_top']
          let topOrBottom = position.up ? 't' : 'b'
          startId = `node-right-${topOrBottom}${props.id}`
        } else {
          v.onStartId = startId = props.onStartParams.id
        }
        createLine({ startId, endId: 'to_top' })
        v.onStartId = startId
      }
      emit('scroll', props.id)
    }

    function onStart(evt, { id, path }) {
      if (!evt.button == 0) return
      const podd = document.getElementById(id)
      const pounds = podd.getBoundingClientRect()

      emit('onStart', { id, path, end: false })
      const oTop = document.getElementById("to_top");

      oTop.style.left = evt.clientX + 'px'
      oTop.style.top = evt.clientY + 'px'

      if (evt.type == 'touchstart') {
        oTop.style.left = evt.targetTouches[0].clientX + 'px'
        oTop.style.top = evt.targetTouches[0].clientY + 'px'
      }

      v.onStartId = id
      createLine({ startId: id, endId: 'to_top' });
      document.ommousewheel = null

      document.onmousemove = function (evt) {
        document.body.style.cursor = 'grabbing'
        var oEvent = evt || window.event;
        var scrollleft = document.documentElement.scrollLeft || document.body.scrollLeft;
        var scrolltop = document.documentElement.scrollTop || document.body.scrollTop;
        oTop.style.left = oEvent.clientX + scrollleft + 0 + "px";
        oTop.style.top = oEvent.clientY + scrolltop + 0 + "px";
        let key = v.onStartId + "~to_top"
        lines[key]?.position()
        evt.preventDefault()
      }

      document.addEventListener('touchmove', ontouchmove, { passive: false })

      function ontouchmove(evt) {
        evt.preventDefault()
        evt.stopPropagation()
        if (!evt.targetTouches) {
          $notify({ type: 'error', title: "have no targetTouches" })
          return
        }
        try {
          var oEvent = evt || window.event;
          var scrollleft = document.documentElement.scrollLeft || document.body.scrollLeft;
          var scrolltop = document.documentElement.scrollTop || document.body.scrollTop;
          oTop.style.left = oEvent.targetTouches[0].clientX + scrollleft + 0 + "px";
          oTop.style.top = oEvent.targetTouches[0].clientY + scrolltop + 0 + "px";
          let key = v.onStartId + "~to_top"
          lines[key]?.position()
        } catch (error) {
          console.error(error);
          $notify({ tpye: "error", text: error })
        }

      }

      document.oncontextmenu = cancelMove

      document.onkeydown = cancelMove
      document.onwheel = onWheel

      function cancelMove(evt) {
        evt.preventDefault()
        document.onmousemove = null
        document.oncontextmenu = null
        document.onmouseup = null
        document.onwheel = null
        document.onkeydown = null
        removeLine(v.onStartId)
        lines = {}
        oTop.style.left = 0
        oTop.style.top = 0
        document.body.style.cursor = ''
        emit('onStart', { id: null, path: null })
        v.onStartId = null
      }

      function onWheel() {
        let key = v.onStartId + "~to_top"
        lines[key]?.position()
        lines[key]?.setOptions({ size: 6 * props.scale })
      }

      document.onmouseup = onmouseup
      document.ontouchend = onmouseup

      function onmouseup(evt) {

        const target = evt.target
        if (evt.type == 'touchend') {
          //鼠标不会触发这个
          let targetElement = document.elementFromPoint(evt.changedTouches[0].clientX, evt.changedTouches[0].clientY);

          let path = targetElement.closest('.path')
          if (path) {
            let endId = path.id.replace('path', '-')
            onEnd({ button: 0 }, endId, null)

          } else {
            cancelMove(evt)
            return
          }
        } else {
          if (evt.button != 0) return
        }

        console.log(target.id);
        console.log(v.onStartId);
        if (target.tagName == 'use') return
        if (target.id == v.onStartId) return;
        let isPath = target.closest('.path')
        if (!target.id || !isPath) {
          cancelMove(evt)
        }
      }
    }

    function onEnd(evt, endId, endPath) {
      document.onmousemove = null
      document.oncontextmenu = null
      document.ontouchend = null
      document.body.style.cursor = '' // 连线结束记得取消鼠标样式
      if (v.onStartId) removeLine(v.onStartId)
      v.onStartId = null
      const oTop = document.getElementById("to_top");
      oTop.style.left = 0
      oTop.style.top = 0
      if (evt.button == 0) {
        emit("dragEnd", { endId, endPath });
      } else {
        emit("dragEnd", {});
      }
    }


    watch(() => props.onStartParams.end, (val) => {
      if (v.onStartId && val) {
        removeLine(v.onStartId)
        v.onStartId = null
      }
    })

    watch(() => props.data.left, (val) => {
      v.left = val
    })
    watch(() => props.data.top, (val) => {
      v.top = val
    })

    watch(() => props.data.disks, async (val) => {
      //  await nextTick() 猜测这个东西可能会引发vue的一些错误 暂时换种做法
      if (props.data.disks.length > 0) {
        setTimeout(() => { getTriggerTarget() }, 300)
      }
    })

    let lines = {};
    function createLine(params) {
      let { startId, endId } = params;
      lines[startId + "~" + endId] = LeaderLine.setLine(
        document.getElementById(startId),
        document.getElementById(endId),
        { dash: { animation: true }, color: "#f38506", size: 6 * props.scale }
      );
      let path = document.querySelector(`#leader-line-${lines[startId + "~" + endId]._id}-line-path`)
      let lineSvg = path.parentNode.parentNode
      lineSvg.style['z-index'] = '9999'
    }

    function removeLine(linename) {
      for (let key in lines) {
        if (key.indexOf(linename) >= 0) {
          const line = lines[key];
          if (line) {
            line.remove();
            delete line[key]
          }
        }
      }
    }

    function handeleMove(e) {
      e.preventDefault();
      if (e.targetTouches?.length > 1) return // 两只手指直接无视
      if (v.saving) return
      if (props.onStartParams.id) return
      store.commit('SET_DRAP', true)
      const target = e.target


      // emit('onMove')
      const node = document.getElementById('node' + props.id)
      // const target = e.target
      let moved = false
      node.startX = e.clientX
      node.startY = e.clientY
      if (e.type == 'touchstart') {
        node.startX = e.targetTouches[0].clientX
        node.startY = e.targetTouches[0].clientY
      }

      target.style.cursor = 'grabbing'
      document.body.style.cursor = 'grabbing'
      document.addEventListener("mousemove", dragMove);
      document.addEventListener("touchmove", touchMove);
      document.addEventListener("mouseup", dragEnd);
      document.addEventListener("touchend", dragEnd);

      function dragMove(e) {
        console.log('moveddddd');
        emit('onMove')
        moved = true
        const offsetX = e.clientX - node.startX;
        const offsetY = e.clientY - node.startY;
        node.startX = e.clientX
        node.startY = e.clientY
        node.style.left = node.offsetLeft + offsetX / props.scale + "px";
        node.style.top = node.offsetTop + offsetY / props.scale + "px";
        e.preventDefault()
      }

      function touchMove(e) {
        console.log('moveddddd');
        emit('onMove')
        moved = true
        const offsetX = e.targetTouches[0].clientX - node.startX;
        const offsetY = e.targetTouches[0].clientY - node.startY;
        node.startX = e.targetTouches[0].clientX
        node.startY = e.targetTouches[0].clientY
        node.style.left = node.offsetLeft + offsetX / props.scale + "px";
        node.style.top = node.offsetTop + offsetY / props.scale + "px";
        e.preventDefault()
      }


      function dragEnd(e) {
        e.preventDefault()
        e.cancelBubble = true
        document.removeEventListener("mousemove", dragMove);
        document.removeEventListener("mouseup", dragEnd);
        document.removeEventListener("touchend", dragEnd);
        document.removeEventListener("touchmove", touchMove);
        target.style.cursor = ''
        document.body.style.cursor = ''
        if (!moved) return
        emit('onMoveEnd')
        v.left = node.style.left
        v.top = node.style.top
        updateNodePosition(node.style.left, node.style.top)
      }
    }

    async function updateNodePosition(left, top) {
      try {
        v.saving = true
        await $axios.put('/api/v1/node', {
          group: props.group, id: props.data.id, left, top, zIndex: v.z
        })
        v.saving = 'saved'
        emit('updatePosition', { index: props.id, left, top })
      } catch (error) {
        console.error(error);
      } finally {
        setTimeout(() => { v.saving = false }, 300)
        store.commit('SET_DRAP', false)
      }
    }

    onBeforeMount(() => {
      if (props.data.left) {
        v.left = props.data.left
        v.top = props.data.top
        if (props.data.saveLeft) {
          updateNodePosition(v.top, v.left)
        }
      }

    })

    let saveTimer = null
    function handleNodeAlive(event) {
      const sel = window.getSelection()
      sel.removeAllRanges()
      if (v.z == store.state.zindex) return
      v.z = store.state.zindex + 1
      store.commit('SET_ZINDEX', v.z)
      // 拖动的自动会保存
      if (saveTimer) clearTimeout(saveTimer)
      saveTimer = setTimeout(() => {
        updateNodeZ()
      }, 500)

    }

    async function updateNodeZ(params) {
      if (v.saving) return
      try {
        await $axios.put('/api/v1/node', {
          group: props.group, id: props.data.id, zIndex: v.z,
          left: v.left, top: v.top
        })
      } catch (error) {
        console.error(error);
      } finally {
        saveTimer = null
      }
    }



    function getTriggerTarget() {
      function getcontent(i) {
        return `${t('node.total')}: ${props.data.disks[i]?.all_space}, ${t('node.free')}: ${props.data.disks[i]?.free_space}`
      }

      v.diskUsage.forEach((target, i) => {
        if (props.data.disks[i]) {
          useTippy(target, {
            triggerTarget: v.diskSpace[i],
            content: computed(() => getcontent(i)),
            theme: 'tooltipdark',
            onShow(instance) {
              return !store.state.isDraging
            },
          });
        }

      })
    }

    function appendTo(t) {
      return document.body
    }

    function onPathMenuShow(instance) {
      v.onShowTippy = instance
      //instance.reference.style.visibility = "visible"
    }

    function onPathMenuHide(instance) {
      v.onShowTippy = {}
      //instance.reference.style.visibility = ''
    }

    const theme = computed(() => store.state.theme);


    onMounted(() => {
      setTimeout(() => {
        getTriggerTarget()
      }, 1000)
    });

    return {
      ...toRefs(v), t,
      theme,
      handlenter,
      handlout,
      handleScroll,
      getTriggerTarget,
      onStart,
      onEnd,
      // handleDiskRepeat,
      handeleMove, handleNodeAlive,
      appendTo,
      onPathMenuShow, onPathMenuHide
    };

  },
};
</script>
<style lang="scss" scoped>
.node-f {
  width: 90vw;
  max-width: 500px;
}

.node {
  background-color: var(--node-bg);
  position: relative;
  border: 1px solid var(--node-bg);
  box-shadow: var(--node-box-shadow);
  border-radius: 2px;
}

.node-info {
  span {
    width: 100%;
  }
}

.disk-f {
  height: 480px;
  overflow-y: auto;
  scrollbar-width: thin;

  &::-webkit-scrollbar {
    width: 6px;
  }

  &::-webkit-scrollbar-track {
    border-radius: 6px;
  }

  &::-webkit-scrollbar-thumb {
    border-radius: 6px;
  }
}

.disk {
  position: relative;
  background-color: var(--disk-bg);

  .diskSpace {
    position: absolute;
    top: 0;
    left: 0;
  }

  .disk-usage {
    height: 4px;
    border-top-right-radius: 2px;
    border-top-left-radius: 2px;
  }
}

.path-f {
  &:hover {
    .path {
      border-color: #f38506 !important;

      .path-dropmenu {
        visibility: visible;
      }
    }

    .podd {
      visibility: visible
    }

    .podd-shadow {
      visibility: visible
    }


  }
}

.path {
  background-color: var(--path-bg);
  border: 2px solid transparent;
  position: relative;

  .w-85 {
    width: 85% !important;
    min-height: 25px;
  }
}

.podd {
  // visibility: hidden;
  width: 14px;
  height: 14px;
  border-radius: 6px;
  transform: translate(-50%, -50%);
  // background-color: pink;
  z-index: 9;
  cursor: grab;

  &:hover {
    background: pink;
  }
}

/* 移动端样式 */
@media only screen and (max-width: 768px) {
  /* 仅在屏幕宽度小于等于768px时生效的样式 */

  .podd-shadow {
    visibility: visible !important;
  }
}


.podd-shadow {
  visibility: hidden;
  width: 14px;
  height: 14px;
  transform: translate(-50%, -50%);
  border: 1px dashed pink;
  border-radius: 50%;
}

.path-dropmenu {
  visibility: hidden;

  &:focus-within {
    visibility: hidden;
  }
}

.lineing {
  background-color: pink;
  visibility: visible;
}

.opacity-50 {
  opacity: 0.5;
}

.opacity-100 {
  opacity: 1;
}

.btn-sm {
  &:focus {
    box-shadow: none;
  }
}

.line-station {
  height: 1px;
  width: 1px;
  // background-color: pink;
}

.cursor-grab {
  cursor: grab
}

.cursor-auto {
  cursor: auto;
}

.disable {
  pointer-events: none;
  opacity: 0.5;
}

.border-heightlight {
  border-color: #f38506;
}

.state-box {
  width: 12px;
  height: 12px;
  border-radius: 50%;
}

.error {
  background: #F00;
  box-shadow: 0 0 3px #f00;
}

.working {
  background: #76ff03;
  box-shadow: 0 0 3px #76ff03;
}

.work-warning {
  background: #ff0;
  box-shadow: 0 0 3px #ff0;
}

.offline {
  background-color: grey;
  box-shadow: rgba(0, 0, 0, 0.2) 0 -1px 7px 1px, inset #002c0b 0 -1px 9px;
}

.bg-green {
  background-color: rgba(115, 191, 105);
}

.bg-red {
  background-color: red;

}
</style>
