<script lang="ts">
import {
  computed,
  defineComponent,
  onBeforeUnmount,
  ref,
  toRefs,
  watch,
  h,
} from 'vue';
import vtkLPSView2DProxy from '@/src/vtk/LPSView2DProxy';
import { useToolStore } from '@/src/store/tools';
import { Tools } from '@/src/store/tools/types';
import useWindowingStore, {
  defaultWindowLevelConfig,
} from '@/src/store/view-configs/windowing';
import { useCurrentImage } from '@/src/composables/useCurrentImage';
import vtkMouseRangeManipulator from '@kitware/vtk.js/Interaction/Manipulators/MouseRangeManipulator';
import { useViewStore } from '@/src/store/views';

function computeStep(min: number, max: number) {
  return Math.min(max - min, 1) / 256;
}

interface Props {
  viewId: string;
}

const WindowLevelToolComponent = defineComponent({
  name: 'WindowLevelTool',
  props: {
    viewId: {
      type: String,
      required: true,
    },
  },
  setup(props) {
    const { viewId: viewID } = toRefs(props);
    const windowingStore = useWindowingStore();
    const viewStore = useViewStore();
    const { currentImageID } = useCurrentImage();

    const viewProxy = computed(
      () => viewStore.getViewProxy<vtkLPSView2DProxy>(viewID.value)!
    );

    const windowConfigDefaults = defaultWindowLevelConfig();
    const wlConfig = computed(() =>
      windowingStore.getConfig(viewID.value, currentImageID.value)
    );

    const wwRange = computed(() => ({
      min: 1e-12, // ensure we don't hit zero and jump to white
      max:
        wlConfig.value != null
          ? wlConfig.value.max - wlConfig.value.min
          : windowConfigDefaults.max,
      step:
        wlConfig.value != null
          ? computeStep(wlConfig.value.min, wlConfig.value.max)
          : computeStep(windowConfigDefaults.min, windowConfigDefaults.max),
      default:
        wlConfig.value != null
          ? wlConfig.value.width
          : windowConfigDefaults.width,
    }));
    const wlRange = computed(() => ({
      min:
        wlConfig.value != null ? wlConfig.value.min : windowConfigDefaults.min,
      max:
        wlConfig.value != null ? wlConfig.value.max : windowConfigDefaults.max,
      step:
        wlConfig.value != null
          ? computeStep(wlConfig.value.min, wlConfig.value.max)
          : computeStep(windowConfigDefaults.min, windowConfigDefaults.max),
      default:
        wlConfig.value != null
          ? wlConfig.value.level
          : windowConfigDefaults.level,
    }));

    const vertVal = ref(0);
    const horizVal = ref(0);

    watch(vertVal, (ww) => {
      if (currentImageID.value !== null) {
        windowingStore.updateConfig(viewID.value, currentImageID.value, {
          width: ww,
        });
      }
    });
    watch(horizVal, (wl) => {
      if (currentImageID.value !== null) {
        windowingStore.updateConfig(viewID.value, currentImageID.value, {
          level: wl,
        });
      }
    });

    const rangeManipulator = vtkMouseRangeManipulator.newInstance({
      button: 1,
      scrollEnabled: false,
    });

    watch(
      viewProxy,
      (curViewProxy, oldViewProxy) => {
        if (oldViewProxy) {
          const istyle = oldViewProxy.getInteractorStyle2D();
          istyle.removeMouseManipulator(rangeManipulator);
        }

        if (curViewProxy) {
          // assumed to be vtkInteractorStyleManipulator
          const istyle = viewProxy.value.getInteractorStyle2D();
          istyle.addMouseManipulator(rangeManipulator);
        }
      },
      { immediate: true }
    );

    onBeforeUnmount(() => {
      // for some reason, VtkTwoView.onBeforeUnmount is being
      // invoked before this onBeforeUnmount during HMR.
      if (!viewProxy.value.isDeleted()) {
        const istyle = viewProxy.value.getInteractorStyle2D();
        istyle.removeMouseManipulator(rangeManipulator);
      }
    });

    function updateManipulator() {
      rangeManipulator.removeAllListeners();
      const vertRange = wwRange.value;
      const horizRange = wlRange.value;

      rangeManipulator.setVerticalListener(
        vertRange.min,
        vertRange.max,
        vertRange.step,
        () => vertVal.value,
        (v) => {
          vertVal.value = v;
        }
      );

      rangeManipulator.setHorizontalListener(
        horizRange.min,
        horizRange.max,
        horizRange.step,
        () => horizVal.value,
        (v) => {
          horizVal.value = v;
        }
      );
    }

    watch(
      () => [wwRange.value, wlRange.value],
      ([ww, wl]) => {
        vertVal.value = ww.default;
        horizVal.value = wl.default;
        updateManipulator();
      },
      { immediate: true }
    );

    return () => null;
  },
});

export default function WindowLevelTool(props: Props) {
  const toolStore = useToolStore();
  const active = computed(() => toolStore.currentTool === Tools.WindowLevel);
  return active.value ? h(WindowLevelToolComponent, props) : [];
}
</script>
