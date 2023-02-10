<template>
  <div id="media-list-view">
    <div class="view-title">
      {{ $T('MEDIA_LIST_PAGE') }} - {{ filterList.length }}
      <el-icon
        style="margin-left: 4px"
        class="cursor-pointer"
        @click="toggleHandleBar"
      >
        <CaretBottom v-show="!handleBarActive" />
        <CaretTop v-show="handleBarActive" />
      </el-icon>
    </div>
    <div
      class="gallery-list"
      :class="{ small: handleBarActive }"
    >
      <div
        v-for="(item, index) in filterList"
        :key="item.id"
        class="gallery-list-item"
      >
        <div
          class="gallery-list__item"
          @click="zoomImage(index)"
        >
          <img
            v-lazy="item.imgUrl"
            class="gallery-list__item-img"
          >
        </div>
        <div
          class="gallery-list__file-name cursor-pointer"
          :title="item.fileName"
          @click="copy(item)"
        >
          {{ item.fileName }}
        </div>
        <div
          class="gallery-list-copy cursor-pointer"
          :title="item.fileName"
          @click="copy(item, true)"
        >
          复制整个URL
        </div>
        <div class="gallery-list-createat">
          {{ dayjs(item.createdAt).format('MM-DD HH:mm:ss') }}
        </div>
        <div class="gallery-list-extname">
          {{ item.extname }}
        </div>
        <div class="gallery-list-size">
          {{ item.width }}x{{ item.height }}
        </div>
        <div class="gallery-list-size">
          <a
            href="#"
            @click.prevent="handleOpenUrl(item.imgUrl)"
          >浏览器打开</a>
        </div>
        <div v-if="false">
          <el-icon
            class="cursor-pointer document"
            @click="copy(item)"
          >
            <Document />
          </el-icon>
          <el-icon
            class="cursor-pointer edit"
            @click="openDialog(item)"
          >
            <Edit />
          </el-icon>
        </div>
      </div>
    </div>
    <div class="block">
      <el-pagination
        v-model:current-page="currentPage"
        :page-sizes="[6,10,20]"
        :page-size="pageSize"
        layout="total, sizes, prev, pager, next"
        :total="images.length"
        @size-change="handleSizeChange"
        @current-change="handleCurrentChange"
      />
    </div>
    <el-dialog
      v-model="dialogVisible"
      :title="$T('CHANGE_IMAGE_URL')"
      width="500px"
      :modal-append-to-body="false"
    >
      <el-input v-model="imgInfo.imgUrl" />
      <template
        #footer
      >
        <el-button @click="dialogVisible = false">
          {{ $T('CANCEL') }}
        </el-button>
        <el-button
          type="primary"
          @click="confirmModify"
        >
          {{ $T('CONFIRM') }}
        </el-button>
      </template>
    </el-dialog>
  </div>
</template>
<script lang="ts" setup>
import dayjs from 'dayjs'
import { PASTE_TEXT, GET_PICBEDS } from '#/events/constants'

import { CaretBottom, Document, Edit, CaretTop } from '@element-plus/icons-vue'
import {
  ipcRenderer,
  IpcRendererEvent,
  shell
} from 'electron'
import { computed, nextTick, onActivated, onBeforeUnmount, onBeforeMount, reactive, ref, watch } from 'vue'
import { getConfig, sendToMain } from '@/utils/dataSender'
import { onBeforeRouteUpdate } from 'vue-router'
import { T as $T } from '@/i18n/index'

import $$db from '@/utils/db'
const images = ref<ImgInfo[]>([])
const dialogVisible = ref(false)
const imgInfo = reactive({
  id: '',
  imgUrl: ''
})
const choosedList: IObjT<boolean> = reactive({})
const gallerySliderControl = reactive({
  visible: false,
  index: 0
})
const choosedPicBed = ref<string[]>([])
const lastChoosed = ref<number>(-1)
const currentPage = ref<number>(1)
const pageSize = ref<number>(10)
const isShiftKeyPress = ref<boolean>(false)
const searchText = ref<string>('')
const handleBarActive = ref<boolean>(false)
const pasteStyle = ref<string>('')

const picBed = ref<IPicBedType[]>([])
onBeforeRouteUpdate((to, from) => {
  if (from.name === 'gallery') {
    clearChoosedList()
  }
  if (to.name === 'gallery') {
    updateGallery()
  }
})

onBeforeMount(async () => {
  ipcRenderer.on('updateGallery', () => {
    nextTick(async () => {
      updateGallery()
    })
  })
  sendToMain(GET_PICBEDS)
  ipcRenderer.on(GET_PICBEDS, getPicBeds)
  updateGallery()

  document.addEventListener('keydown', handleDetectShiftKey)
  document.addEventListener('keyup', handleDetectShiftKey)
})

function handleDetectShiftKey (event: KeyboardEvent) {
  if (event.key === 'Shift') {
    if (event.type === 'keydown') {
      isShiftKeyPress.value = true
    } else if (event.type === 'keyup') {
      isShiftKeyPress.value = false
    }
  }
}

const filterList = computed(() => {
  return getGallery()
})

function getPicBeds (event: IpcRendererEvent, picBeds: IPicBedType[]) {
  picBed.value = picBeds
}

function getGallery (): IGalleryItem[] {
  if (searchText.value || choosedPicBed.value.length > 0) {
    return images.value
      .filter(item => {
        let isInChoosedPicBed = true
        let isIncludesSearchText = true
        if (choosedPicBed.value.length > 0) {
          isInChoosedPicBed = choosedPicBed.value.some(type => type === item.type)
        }
        if (searchText.value) {
          isIncludesSearchText = item.fileName?.includes(searchText.value) || false
        }
        return isIncludesSearchText && isInChoosedPicBed
      }).map(item => {
        return {
          ...item,
          src: item.imgUrl || '',
          key: (item.id || `${Date.now()}`),
          intro: item.fileName || ''
        }
      })
  } else {
    const start = (pageSize.value) * (currentPage.value - 1)
    const end = (pageSize.value) * currentPage.value
    return images.value.slice(start, end).map(item => {
      return {
        ...item,
        src: item.imgUrl || '',
        key: (item.id || `${Date.now()}`),
        intro: item.fileName || ''
      }
    })
  }
}

async function updateGallery () {
  images.value = (await $$db.get({ orderBy: 'desc' })).data
}

watch(() => filterList, () => {
  clearChoosedList()
})

function clearChoosedList () {
  isShiftKeyPress.value = false
  Object.keys(choosedList).forEach(key => {
    choosedList[key] = false
  })
  lastChoosed.value = -1
}

function zoomImage (index: number) {
  gallerySliderControl.index = index
  gallerySliderControl.visible = true
  changeZIndexForGallery(true)
}

function changeZIndexForGallery (isOpen: boolean) {
  if (isOpen) {
    // @ts-ignore
    document.querySelector('.main-content.el-row').style.zIndex = 101
  } else {
    // @ts-ignore
    document.querySelector('.main-content.el-row').style.zIndex = 10
  }
}

async function copy (item: ImgInfo, copyFull: boolean = false) {
  const copyLink = await ipcRenderer.invoke(PASTE_TEXT, item, true, copyFull)
  const obj = {
    title: $T('COPY_LINK_SUCCEED'),
    body: copyLink
    // sometimes will cause lagging
    // icon: item.url || item.imgUrl
  }
  const myNotification = new Notification(obj.title, obj)
  myNotification.onclick = () => {
    return true
  }
}

function openDialog (item: ImgInfo) {
  imgInfo.id = item.id!
  imgInfo.imgUrl = item.imgUrl as string
  dialogVisible.value = true
}

async function confirmModify () {
  await $$db.updateById(imgInfo.id, {
    imgUrl: imgInfo.imgUrl
  })
  const obj = {
    title: $T('CHANGE_IMAGE_URL_SUCCEED'),
    body: imgInfo.imgUrl
    // icon: this.imgInfo.imgUrl
  }
  const myNotification = new Notification(obj.title, obj)
  myNotification.onclick = () => {
    return true
  }
  dialogVisible.value = false
  updateGallery()
}

// function choosePicBed (type: string) {
//   const idx = choosedPicBed.value.indexOf(type)
//   if (idx !== -1) {
//     choosedPicBed.value.splice(idx, 1)
//   } else {
//     choosedPicBed.value.push(type)
//   }
// }

function toggleHandleBar () {
  handleBarActive.value = !handleBarActive.value
}

onBeforeUnmount(() => {
  ipcRenderer.removeAllListeners('updateGallery')
  ipcRenderer.removeListener(GET_PICBEDS, getPicBeds)
})

onActivated(async () => {
  pasteStyle.value = (await getConfig('settings.pasteStyle')) || 'markdown'
})

function handleSizeChange (val) {
  console.log(`${val} items per page`)
}

function handleCurrentChange (val) {
  console.log(`current page: ${val}`)
}

function handleOpenUrl (url?: string) {
  url && shell.openExternal(url)
}
</script>
<script lang="ts">
export default {
  name: 'GalleryPage'
}
</script>
<style lang='stylus'>
.PhotoSlider
  &__BannerIcon
    &:nth-child(1)
      display none
  &__Counter
    margin-top 20px
.view-title
  color #eee
  font-size 20px
  text-align center
  margin 10px auto
  .sub-title
    font-size 14px
  .el-icon-caret-bottom
    cursor: pointer
    transition all .2s ease-in-out
    &.active
      transform: rotate(180deg)
#gallery-view
  height 100%
  .cursor-pointer
    cursor pointer
.item-base
  background #2E2E2E
  text-align center
  padding 5px 0
  cursor pointer
  font-size 13px
  transition all .2s ease-in-out
  height: 28px
  box-sizing: border-box
  &.copy
    cursor not-allowed
    background #49B1F5
    &.active
      cursor pointer
      background #1B9EF3
      color #fff
  &.delete
    cursor not-allowed
    background #F47466
    &.active
      cursor pointer
      background #F15140
      color #fff
  &.all-pick
    cursor not-allowed
    background #69C282
    &.active
      cursor pointer
      background #44B363
      color #fff
#media-list-view
  position relative
  display flex
  flex-direction column
  height 100%
  .round
    border-radius 14px
  .pull-right
    float right
  .gallery-list
    flex  1
    box-sizing border-box
    padding 8px 0
    padding-left 20px
    overflow-y auto
    overflow-x hidden
    transition all .2s ease-in-out .1s
    width 100%
    color #fff
    .gallery-list-item
      margin-bottom 4px
      height 100px
      display flex
      > div
        padding 0 10px
    &.small
      height: 287px
      top: 113px

    &__item
      flex 1
      transition all .2s ease-in-out
      cursor pointer
      overflow hidden
      display flex
      margin-bottom 6px
      color #fff
      &-fake
        position absolute
        top 0
        left 0
        opacity 0
        width 100%
        z-index -1
      &:hover
        transform scale(1.1)
      &-img
        // width 100%
        object-fit cover
        min-width 50%
    &__tool-panel
      color #ddd
      margin-bottom 4px
      display flex
      .el-checkbox
        height 16px
      i
        cursor pointer
        transition all .2s ease-in-out
        margin-right 4px
        &.document
          &:hover
            color #49B1F5
        &.edit
          &:hover
            color #69C282
        &.delete
          &:hover
            color #F15140
    &__file-name
      width 150px
      word-break: break-word;
      color #ddd
      font-size 14px
    .gallery-list-copy
      width 90px
    .gallery-list-copy
    .gallery-list-createat
    .gallery-list-extname
    .gallery-list-size
      font-size 14px
    .gallery-list-extname
      flex-shrink none
  .handle-bar
    color #ddd
    margin-bottom 10px
</style>
