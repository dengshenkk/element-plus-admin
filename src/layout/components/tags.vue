<template>
    <div class='position'>
        <el-scrollbar
            ref='scrollbar'
            wrap-class='scrollbar-wrapper'
        >
            <div class='layout-tags-container whitespace-nowrap'>
                <span
                    v-for='v in tagsList'
                    :key='v.path'
                    :ref='getTagsDom'
                    class='border border-gray-200 px-2 py-1 mx-1 cursor-pointer rounded-md'
                    :class='{"layout-tags-active": v.isActive}'
                    @contextmenu.prevent='contextRightMenu(v,$event)'
                >
                    <i
                        v-if='v.isActive'
                        class='rounded-full inline-block w-2 h-2 bg-white -ml-1 mr-1'
                    />
                    <router-link :to='v.path'>{{ v.title }}</router-link>
                    <i
                        v-if='tagsList.length>1'
                        class='el-icon-close text-xs hover:bg-gray-300 hover:text-white rounded-full leading-3 p-0.5 ml-1 -mr-1'
                        @click='removeTagNav(v)'
                    />
                </span>
            </div>
        </el-scrollbar>
        <ul
            ref='rightMenuEl'
            class='border border-gray-200 absolute w-24 leading-none bg-white shadow-md rounded-lg py-0.5 z-10'
            :style='menuPos'
        >
            <li
                class='px-4 py-2 cursor-pointer hover:bg-gray-200'
                @click='refresh'
            >
                刷新
            </li>
            <li
                class='px-4 py-2 cursor-pointer hover:bg-gray-200'
                @click='closeOther'
            >
                关闭其它
            </li>
            <li
                class='px-4 py-2 cursor-pointer hover:bg-gray-200'
                @click='closeAll'
            >
                关闭所有
            </li>
        </ul>
    </div>
</template>

<script lang="ts">
import { defineComponent, nextTick, ref, watch, onBeforeUpdate, onMounted, reactive, Ref, ComponentInternalInstance } from 'vue'
import { useStore } from '/@/store/index'
import { Store } from 'vuex'
import { useRoute, useRouter, Router, RouteLocationNormalizedLoaded } from 'vue-router'
import { ITagsList } from '/@/type/store/layout'

// 右键菜单
const rightMenu = (store:Store<IState>, router: Router, route: RouteLocationNormalizedLoaded) => {
    const menuPos = reactive({
        left: '0px',
        top: '0px',
        display: 'none'
    })
    const rightMenuEl:Ref<HTMLElement | null> = ref(null)
    // 当前右键的那个标签
    let currentRightTags:ITagsList
    const contextRightMenu = (v:ITagsList, event:MouseEvent) => {
        currentRightTags = v
        menuPos.display = 'block'
        nextTick(() => {
            let left = event.clientX - 5
            if(!rightMenuEl.value) return
            if(event.clientX + rightMenuEl.value.offsetWidth > document.body.offsetWidth) {
                left = event.clientX - rightMenuEl.value.offsetWidth
            }
            menuPos.left = `${left}px`
            menuPos.top = `${event.clientY + 10}px`
        })
    }
    const refresh = () => {
        if(currentRightTags.path === route.path) {
            router.replace(`/redirect${currentRightTags.path}`)
        }else{
            router.push(`/redirect${currentRightTags.path}`)
        }
    }
    const closeOther = () => store.commit('layout/removeOtherTagNav', currentRightTags)
    document.body.addEventListener('click', () => menuPos.display = 'none')
    return { menuPos, contextRightMenu, refresh, rightMenuEl, closeOther }
}

// 标签页滚动
const tagScroll = (store:Store<IState>) => {
    const { tagsList, cachedViews } = store.state.layout.tags
    const scrollbar:Ref<{wrap:HTMLElement, update():void} | null> = ref(null)
    const layoutTagsItem:Ref<Array<ComponentInternalInstance | Element | null>> = ref([])
    const getTagsDom = (el:ComponentInternalInstance | Element | null) => el && layoutTagsItem.value.push(el)
    // 监听标签页导航
    watch(
        () => tagsList.length,
        () => nextTick(() => {
            if(!scrollbar.value) return
            scrollbar.value.update()
            nextTick(() => {
                const itemWidth = layoutTagsItem.value.filter(v => v).reduce((acc, v) => {
                    const val = v as HTMLElement
                    return acc + val.offsetWidth + 6
                }, 0)
                if(!scrollbar.value) return
                const scrollLeft = itemWidth - scrollbar.value.wrap.offsetWidth + 70
                if(scrollLeft > 0) scrollbar.value.wrap.scrollLeft = scrollLeft
            })
        })
    )
    // 确保在每次变更之前重置引用
    onBeforeUpdate(() => {
        layoutTagsItem.value = []
    })
    return { tagsList, scrollbar, layoutTagsItem, cachedViews, getTagsDom }
}
export default defineComponent({
    name: 'LayoutTags',
    setup() {
        const store = useStore()
        const route = useRoute()
        const router = useRouter()
        const removeTagNav = (v: any) => store.commit('layout/removeTagNav', { cPath: route.path, tagsList: v })
        const closeAll = () => store.commit('layout/removeAllTagNav')
        
        onMounted(() => {
            store.commit('layout/addCachedViews', { name: route.name, noCache: route.meta.noCache })
        })
        
        const rightMenuData = rightMenu(store, router, route)
        const tagScrollData = tagScroll(store)
        
        return {
            removeTagNav,
            ...tagScrollData,
            ...rightMenuData,
            closeAll
        }
    }
})
</script>