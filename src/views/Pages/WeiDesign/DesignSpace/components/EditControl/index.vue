<template>
    <div class="edit-control-container">
        <div class="wrap-container">
            <div
                id="wrap"
                @scroll.prevent="scrollEdit"
                @mousedown.prevent="wrapMousedown"
                @mousewheel="mouseWheel"
                ref="$wrap"
            >
                <div id="content">
                    <div
                        ref="canvasRef"
                        class="edit-canvas"
                        @mousemove="canvasMousemove"
                        :style="{transform: `scale(${ scaleValueReal })`, cursor: isEnterSpace ? 'pointer' : 'auto', ...pageConfig}"
                        @drop="handleDrop"
                        @dragover="handleDragOver"
                        v-loading="!componentData.length"
                    >
                        <div class="components-show-content">
                            <!--页面组件列表展示-->
                            <Shape
                                v-for="(item, index) in componentData"
                                :defaultStyle="item.style"
                                :style="item.style"
                                :key="item.id + item.id"
                                :element="item"
                                :zIndex="index"
                                :index="index"
                            >
                                <component
                                    class="custom-component-class"
                                    :is="item.component"
                                    :chartOption="item.chartOption"
                                />
                            </Shape>
                            <MarkLine></MarkLine>
                        </div>
                    </div>
                </div>
            </div>
            <div class="edit-bottom-menu">
                <span class="key-down-show">按下 [ {{}} ] 键</span>
                <el-slider
                    v-model="sliderConfig.scaleValue"
                    :format-tooltip="sliderConfig.formatSliderTip"
                    @input="sliderConfig.inputScale"
                    show-input
                    size="small"
                />
            </div>
        </div>
        <SketchRule
            :key="sketchRuleKey"
            ref="$sketchRule"
            class="ruler-container"
            :lang="lang"
            :thick="thick"
            :scale="scaleValueReal"
            :width="10000"
            :height="10000"
            :startX="-5000 / scaleValueReal"
            :startY="-5000 / scaleValueReal"
            :shadow="shadow"
            :horLineArr="lines.h"
            :verLineArr="lines.v"
            :cornerActive="true"
            @handleLine="handleLine"
            @onCornerClick="handleCornerClick"
        />
    </div>
</template>

<script lang="ts" setup>
import { reactive, ref, onMounted, watch, computed, watchEffect, nextTick } from "vue";
import { useDesignStore } from '@/stores/design';
import {deepCopy, uuid, throttle, debounce} from "@/utils";

import SketchRule from "@/components/Ruler/sketchRuler.vue";
import Shape from '@/components/Editor/Shape.vue';
import MarkLine from '@/components/Editor/MarkLine.vue';
import { useResizeObserver } from "@vueuse/core";
import { addDesign, setImg } from '@/api/service/design'
import { useRoute } from "vue-router";
import { useHtml2canvas } from "@/hooks/useDom2image";
import { dataURItoBlob } from "@/utils";
import { ElMessage } from 'element-plus'

const store = useDesignStore();
const route = useRoute();

const $wrap = ref<any>();
const $sketchRule = ref<any>();
const canvasRef = ref<any>();
const sketchRuleKey =ref<string>('');
const domeStr = ref<string>('');
const pageRef = ref();

// 缩放可视区
const sliderConfig: any = reactive({
    scaleValue: 60,
    inputScale: (scale: number) => {
        sliderConfig.scaleValue = scale;
        sketchRuleKey.value = (new Date).toString();
    },
    formatSliderTip: (value: any) => {
        return value + '%'
    }
});
const scaleValueReal = computed(() => {
    return sliderConfig.scaleValue / 100
});

const lines = reactive({
    h: [],
    v: [],
});
const thick = ref(20);
const lang = ref("zh-CN");
const shadow = reactive({
    x: 0,
    y: 0,
    width: 0,
    height: 0,
});

const handleLine = (e: any) => {
    console.log(e)
};

const handleCornerClick = (e: any) => {
    console.log(e);
};

const hRulerX = ref("0");
const hRulerY = ref("0");
const scrollEdit = (e: any) => {
    hRulerY.value = '-' + e.target.scrollTop + 'px';
    hRulerX.value = '-' + e.target.scrollLeft + 'px';
}



const startMoveWrap = reactive({
    x: 0,
    y: 0
})

const wrapMousedown = (e: any) => {
    if(e.target && e.target.id === 'content' || isEnterSpace.value) {
        startMoveWrap.x = e.x;
        startMoveWrap.y = e.y;

        document.onmousemove = (e: any) => {
            $wrap.value.scrollLeft =  $wrap.value.scrollLeft - (e.x - startMoveWrap.x);
            $wrap.value.scrollTop = $wrap.value.scrollTop - (e.y - startMoveWrap.y);
            hRulerY.value = '-' + $wrap.value.scrollTop + 'px';
            hRulerX.value = '-' + $wrap.value.scrollLeft + 'px';
            startMoveWrap.x = e.x;
            startMoveWrap.y = e.y;
        }

        document.onmouseup = () => {
            document.onmousemove=null;
            document.onmouseup=null;
        }
    }
}

const mouseWheel = (e: any) => {
    if( isEnterSpace.value) {
        let mouseTo = e && (e.deltaY > 0 || e.deltaX > 0) ? 'down' : 'up';
        if(mouseTo === 'down') {
            sliderConfig.scaleValue = sliderConfig.scaleValue - 5;
        }else {
            sliderConfig.scaleValue = sliderConfig.scaleValue + 5
        }
    }
}

/**
 * 设置wrap显示的位置和大小
 */
const setWrapPositionSize = () => {
    // 监听wrap的尺寸变化
    useResizeObserver($wrap, debounce(function() {
        const wrapW = $wrap.value.clientWidth;
        const wrapH = $wrap.value.clientHeight;
        const canvasW = canvasRef.value.clientWidth;
        const canvasH = canvasRef.value.clientHeight;
        if(canvasW > canvasH) {
            sliderConfig.scaleValue = ~~(((wrapW -200) / canvasW) * 100);  // 数字取整
        }else {
            sliderConfig.scaleValue = ~~(((wrapH -200) / canvasH) * 100);  // 数字取整
        }
        const scale = sliderConfig.scaleValue / 100;
        const x = ($wrap.value.clientWidth - canvasRef.value.clientWidth * scale) / 2;
        const y = ($wrap.value.clientHeight - canvasRef.value.clientHeight * scale) / 2;
        $wrap.value.scrollTop = 5000 - y;
        $wrap.value.scrollLeft = 5000 - x;
        hRulerY.value = '-' + $wrap.value.scrollTop + 'px';
        hRulerX.value = '-' + $wrap.value.scrollLeft + 'px';
    }, 50));
}

defineExpose({
    setWrapPositionSize
})

// 监听键盘按键事件componentData
const isEnterSpace = ref(false);
const keyEvent = () => {
    document.addEventListener('keydown', (e: any) => {
        if(e && e.code === 'Space') {
            isEnterSpace.value = true;
            e.preventDefault(); // 阻止默认事件行为
        }
    });
    document.addEventListener('keyup', (e: any) => {
        if(e && e.code === 'Space') {
            isEnterSpace.value = false;
            e.preventDefault(); // 阻止默认事件行为
        }
    });
}

onMounted(() => {
    setWrapPositionSize();
    keyEvent();
});

// 自定义组件
const componentData = computed(() => store.$state.componentsInCanvas);

// 拖拽组件到当前画布
const handleDrop = (e: any) => {
    e.preventDefault();
    e.stopPropagation();

    const component = deepCopy(JSON.parse(e.dataTransfer.getData('component')) || {});
    const x = e.offsetX;
    const y = e.offsetY;
    const width = parseInt(component.style.width || 0);
    const height = parseInt(component.style.height || 0);
    component.style.top = y - height / 2 + 'px';
    component.style.left = x - width / 2 + 'px';
    component.style.zIndex = componentData.value.length + 1; // 组件的定位层级
    component.id = uuid(); // 生成uuid
    component.ifLock = false; // 是否锁定
    component.ifShow = true; // 是否显示
    component.title = `${component.label}-${componentData.value.length + 1}`;
    store.addComponentsInCanvas(component);
}
const handleDragOver = (e: any) => {
    e.preventDefault();
}

watch(() => sliderConfig.scaleValue, (n) => {
    sliderConfig.scaleValue = n < 10 ? 10 : n;
    store.$patch({
        canvasScale: scaleValueReal.value
    })
})

const canvasMousemove = () => {
    // const { x, y } = useMouseXY();
    // console.log(x, y)
}

// page 配置变动
const pageConfig = computed(() => {
    const pageConfig = computed(() => store.$state.pageConfig);
    return {
        width: pageConfig.value.width + 'px',
        height: pageConfig.value.height + 'px',
        backgroundColor: pageConfig.value.backgroundColor
    }
});

// 不适用深层监听 性能消耗
const editConfigContent = computed(() => store.editConfigContent);

// 数据变动更新
const setComponentsUpdate = async () => {
    console.log('配置更新');

    const res = await useHtml2canvas(canvasRef.value as HTMLElement, {});
    const content = JSON.stringify(editConfigContent.value);
    await store.updateDesignById(canvasId, content);
    await setImg(canvasId, res);
}

const canvasId = String(route.query?.key) || '';

store.getEditConfigContent(canvasId).then(res => {
    watch([componentData, () => store.$state.pageConfig], debounce(async function() {
    await setComponentsUpdate();
}, 1000, false), { deep: true })
});
</script>

<style lang="scss" scoped>
.edit-control-container {
    width: 100%;
    height: 100%;
    background-image: linear-gradient(#fafafc 14px,transparent 0),linear-gradient(90deg,transparent 14px,#373739 0);
    background-color: #fff;
    background-size: 15px 15px,15px 15px;
    position: relative;
    overflow: hidden;

    .wrap-container {
        width: 100%;
        height: 100%;
        position: absolute;
        overflow: hidden;

        #wrap {
            position: absolute;
            width: 100%;
            height: calc(100% - 40px);
            user-select: none;
            padding-bottom: 0;
            top: 0;
            overflow: auto;

            // &:hover {
            //     overflow: auto;
            // }

            #content {
                width: 10000px;
                height: 10000px;
                position: absolute;
                top: 0;
                left: 0;

                .edit-canvas {
                    height: v-bind('pageConfig.height')px;
                    width: v-bind('pageConfig.width')px;
                    position: absolute;
                    background-color: v-bind('pageConfig.backgroundColor');
                    top: 50%;
                    left: 50%;
                    box-shadow: 0 8px 10px #00000012;
                    border-radius: 20px;
                    -webkit-transform-origin: 0 0;
                    transform-origin: 0 0;
                    overflow: hidden;

                    .components-show-content {
                        height: v-bind('pageConfig.height')px;
                        width: v-bind('pageConfig.width')px;

                        .custom-component-class {
                            width: 100%;
                            height: 100%;
                            pointer-events: none;
                        }

                    }
                }
            }
        }

        .edit-bottom-menu {
            width: 100%;
            height: 40px;
            background-color: $bgColor;
            z-index: 100;
            position: absolute;
            bottom: 0;
            display: flex;
            align-items: center;
            padding: 10px;

            .el-slider {
                width: 300px;
                float: right;
                margin-left: auto;
            }

            .key-down-show {
                font-weight: bold;
            }
        }

    }


}
</style>

<style lang="scss">
.demos-img {
    position: absolute;
    top: 0;
    left: 0;
    width: 192px;
    height: 108px;
}
.ruler-container {

    .h-container {
        position: absolute;
        width: 10000px!important;
        top: 0;
        left: v-bind(hRulerX)!important;
        transform-origin: center;
    }

    .v-container {
        position: absolute;
        height: 10000px!important;
        left: 0;
        top: v-bind(hRulerY)!important;
        transform-origin: center;
    }
}
</style>

