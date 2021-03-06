<template>
  <ScrollContainer ref="wrapperRef" :style="wrapStyle">
    <div ref="spinRef" :style="spinStyle" v-loading="loading" :loading-tip="loadingTip">
      <slot />
    </div>
  </ScrollContainer>
</template>
<script lang="ts">
  import type { ModalWrapperProps } from '../types';
  import type { CSSProperties } from 'vue';

  import {
    defineComponent,
    computed,
    ref,
    watchEffect,
    unref,
    watch,
    onMounted,
    nextTick,
    onUnmounted,
  } from 'vue';
  import { Spin } from 'ant-design-vue';

  import { useWindowSizeFn } from '/@/hooks/event/useWindowSizeFn';
  import { ScrollContainer } from '/@/components/Container';

  // import { useElResize } from '/@/hooks/event/useElResize';
  import { propTypes } from '/@/utils/propTypes';
  import { createModalContext } from '../hooks/useModalContext';

  export default defineComponent({
    name: 'ModalWrapper',
    components: { Spin, ScrollContainer },
    props: {
      loading: propTypes.bool,
      useWrapper: propTypes.bool.def(true),
      modalHeaderHeight: propTypes.number.def(50),
      modalFooterHeight: propTypes.number.def(54),
      minHeight: propTypes.number.def(200),
      footerOffset: propTypes.number.def(0),
      visible: propTypes.bool,
      fullScreen: propTypes.bool,
      loadingTip: propTypes.string,
    },
    emits: ['height-change', 'ext-height'],
    setup(props: ModalWrapperProps, { emit }) {
      const wrapperRef = ref<ComponentRef>(null);
      const spinRef = ref<ElRef>(null);
      const realHeightRef = ref(0);
      const minRealHeightRef = ref(0);

      let stopElResizeFn: Fn = () => {};

      useWindowSizeFn(setModalHeight);

      createModalContext({
        redoModalHeight: setModalHeight,
      });

      const wrapStyle = computed(
        (): CSSProperties => {
          return {
            minHeight: `${props.minHeight}px`,
            height: `${unref(realHeightRef)}px`,
            // overflow: 'auto',
          };
        }
      );

      const spinStyle = computed(
        (): CSSProperties => {
          return {
            // padding 28
            height: `${unref(realHeightRef) - 28}px`,
          };
        }
      );

      watchEffect(() => {
        props.useWrapper && setModalHeight();
      });

      watch(
        () => props.fullScreen,
        (v) => {
          setModalHeight();
          if (!v) {
            realHeightRef.value = minRealHeightRef.value;
          } else {
            minRealHeightRef.value = realHeightRef.value;
          }
        }
      );

      onMounted(() => {
        const { modalHeaderHeight, modalFooterHeight } = props;
        emit('ext-height', modalHeaderHeight + modalFooterHeight);
        // listenElResize();
      });

      onUnmounted(() => {
        stopElResizeFn && stopElResizeFn();
      });

      async function setModalHeight() {
        // 解决在弹窗关闭的时候监听还存在,导致再次打开弹窗没有高度
        // 加上这个,就必须在使用的时候传递父级的visible
        if (!props.visible) return;
        const wrapperRefDom = unref(wrapperRef);
        if (!wrapperRefDom) return;
        const bodyDom = wrapperRefDom.$el.parentElement;
        if (!bodyDom) return;
        bodyDom.style.padding = '0';
        await nextTick();

        try {
          const modalDom = bodyDom.parentElement && bodyDom.parentElement.parentElement;
          if (!modalDom) return;

          const modalRect = getComputedStyle(modalDom).top;
          const modalTop = Number.parseInt(modalRect);
          let maxHeight =
            window.innerHeight -
            modalTop * 2 +
            (props.footerOffset! || 0) -
            props.modalFooterHeight -
            props.modalHeaderHeight;

          // 距离顶部过进会出现滚动条
          if (modalTop < 40) {
            maxHeight -= 26;
          }
          await nextTick();
          const spinEl = unref(spinRef);

          if (!spinEl) return;

          const realHeight = spinEl.scrollHeight;

          if (props.fullScreen) {
            realHeightRef.value =
              window.innerHeight - props.modalFooterHeight - props.modalHeaderHeight;
          } else {
            realHeightRef.value = realHeight > maxHeight ? maxHeight : realHeight + 16 + 30;
          }
          emit('height-change', unref(realHeightRef));
        } catch (error) {
          console.log(error);
        }
      }

      return { wrapStyle, wrapperRef, spinRef, spinStyle };
    },
  });
</script>
