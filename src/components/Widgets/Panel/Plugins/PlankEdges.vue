<template>
  <div>
  </div>
</template>

<script>
import { Vector3 } from 'three';
import EventBus from '../../../EventBus/EventBus';

export default {
  name: 'PlankEdges',
  inject: ['vglNamespace'],
  props: {
    plankPosition: {
      type: Object,
      required: true,
    },
    plankDimension: {
      type: Object,
      required: true,
    },
    plankType: {
      type: String,
      required: true,
    },
    plankEdges: {
      type: String,
      required: true,
    },
    plankPoints: {
      type: Array,
      required: false,
    },
    x: {
      type: Number,
      required: true,
    },
    y: {
      type: Number,
      required: true,
    },
  },
  data() {
    return {
      topElement: null,
      leftElement: null,
      rightElement: null,
      baseElement: null,
      edgeElements: [],
      containerElement: null,
      showAllEdges: false,
    };
  },
  computed: {
    domElement() {
      return this.vglNamespace.renderers[0].inst.domElement;
    },
    cameraInst() {
      return this.vglNamespace.cameras.camera1;
    },
    edges: {
      get() {
        return this.plankEdges.split('-');
      },
      set(val) {
        // val [one, two, three, four]
        this.$emit('update:plankEdges', val.join('-'));
        this.$nextTick(() => EventBus.$emit('save'));
      },
    },
  },
  mounted() {
    this.addElementsToDOM();
    this.changeEventHandler(true);
    this.vglNamespace.beforeRender.push(this.updateStyle);
  },
  beforeDestroy() {
    this.changeEventHandler(false);
    this.vglNamespace.beforeRender.splice(this.vglNamespace.beforeRender.indexOf(this.updateStyle), 1);
    this.removeElements();
  },
  methods: {
    projectVectorTo2D(x, y, z) {
      const p = new Vector3(x, y, z);
      const vector = p.project(this.cameraInst);
      vector.x = (vector.x + 1) / 2 * this.domElement.width - 25;
      vector.y = -(vector.y - 1) / 2 * this.domElement.height - 10;

      const rect = this.domElement.getBoundingClientRect();
      vector.x = Math.max(30, Math.min(vector.x, rect.width - 70));
      vector.y = Math.max(30, Math.min(vector.y, rect.height - 50));

      return vector;
    },
    createSelect() {
      const select = document.createElement('select');
      const option1 = document.createElement('option');
      const option2 = document.createElement('option');
      const option3 = document.createElement('option');
      option1.setAttribute('value', '0');
      option2.setAttribute('value', '6');
      option3.setAttribute('value', '20');
      option1.innerHTML = 'non';
      option2.innerHTML = '0.6';
      option3.innerHTML = '20';
      select.appendChild(option1);
      select.appendChild(option2);
      select.appendChild(option3);
      select.classList.add('plank-edge-input');

      return select;
    },
    updateStyle() {
      const {
        plankPosition,
        plankDimension,
        plankPoints,
        plankType,
      } = this;

      if (plankPoints) {
        const borderedEdges = new Array(4);
        const borderedCount = new Array(4);
        const edgePositions = new Array(4);

        plankPoints.forEach((point, index) => {
          const nextIndex = (index + 1) % plankPoints.length;
          let edgePosition = new Vector3(plankPosition.x, plankPosition.y, plankPosition.z);

          if (plankType === 'FP') {
            edgePosition = new Vector3(
              plankPosition.x - plankDimension.width / 2 + (point[1] + plankPoints[nextIndex][1]) * 5,
              plankPosition.y,
              plankPosition.z - plankDimension.depth / 2 + (point[0] + plankPoints[nextIndex][0]) * 5,
            );
          } else if (plankType === 'VP') {
            edgePosition = new Vector3(
              plankPosition.x - plankDimension.width / 2 + (point[0] + plankPoints[nextIndex][0]) * 5,
              plankPosition.y - plankDimension.height / 2 + (point[1] + plankPoints[nextIndex][1]) * 5,
              plankPosition.z,
            );
          } else if (plankType === 'VDP') {
            edgePosition = new Vector3(
              plankPosition.x,
              plankPosition.y - plankDimension.height / 2 + (point[1] + plankPoints[nextIndex][1]) * 5,
              plankPosition.z - plankDimension.depth / 2 + (point[0] + plankPoints[nextIndex][0]) * 5,
            );
          }

          const vectorEdge2D = this.projectVectorTo2D(edgePosition.x, edgePosition.y, edgePosition.z);

          this.edgeElements[index].style.top = `${vectorEdge2D.y}px`;
          this.edgeElements[index].style.left = `${vectorEdge2D.x}px`;

          if (!this.showAllEdges) {
            const borderNumber = this.borderedEdgeID(point, plankPoints[nextIndex]);
            if (borderNumber > 0 && !borderedEdges[borderNumber - 1]) {
              borderedEdges[borderNumber - 1] = index;
              borderedCount[borderNumber - 1] = 1;
              edgePositions[borderNumber - 1] = edgePosition;
              this.edgeElements[index].style.display = 'unset';
            } else {
              if (borderNumber > 0) {
                borderedCount[borderNumber - 1] += 1;
                edgePositions[borderNumber - 1].add(edgePosition);
              }
              this.edgeElements[index].style.display = 'none';
            }
          }
        });

        if (!this.showAllEdges) {
          borderedEdges.forEach((edge, index) => {
            edgePositions[index].divideScalar(borderedCount[index]);
            const vectorEdge2D = this.projectVectorTo2D(edgePositions[index].x, edgePositions[index].y, edgePositions[index].z);

            this.edgeElements[edge].style.top = `${vectorEdge2D.y}px`;
            this.edgeElements[edge].style.left = `${vectorEdge2D.x}px`;
          });
        }
      } else {
        const topPosition = new Vector3(
          plankPosition.x,
          plankType === 'VDP' || plankType === 'VP' ? plankPosition.y + plankDimension.height / 2 : plankPosition.y,
          plankType === 'FP' ? plankPosition.z - plankDimension.depth / 2 : plankPosition.z,
        );
        const vectorTop2D = this.projectVectorTo2D(topPosition.x, topPosition.y, topPosition.z);
        this.topElement.style.top = `${vectorTop2D.y}px`;
        this.topElement.style.left = `${vectorTop2D.x}px`;

        const leftPosition = new Vector3(
          plankType === 'FP' || plankType === 'VP' ? plankPosition.x - plankDimension.width / 2 : plankPosition.x,
          plankPosition.y,
          plankType === 'VDP' ? plankPosition.z + plankDimension.depth / 2 : plankPosition.z,
        );
        const vectorLeft2D = this.projectVectorTo2D(leftPosition.x, leftPosition.y, leftPosition.z);
        this.leftElement.style.top = `${vectorLeft2D.y}px`;
        this.leftElement.style.left = `${vectorLeft2D.x}px`;

        const rightPosition = new Vector3(
          plankType === 'FP' || plankType === 'VP' ? plankPosition.x + plankDimension.width / 2 : plankPosition.x,
          plankPosition.y,
          plankType === 'VDP' ? plankPosition.z - plankDimension.depth / 2 : plankPosition.z,
        );
        const vectorRight2D = this.projectVectorTo2D(rightPosition.x, rightPosition.y, rightPosition.z);
        this.rightElement.style.top = `${vectorRight2D.y}px`;
        this.rightElement.style.left = `${vectorRight2D.x}px`;

        const basePosition = new Vector3(
          plankPosition.x,
          plankType === 'VDP' || plankType === 'VP' ? plankPosition.y - plankDimension.height / 2 : plankPosition.y,
          plankType === 'FP' ? plankPosition.z + plankDimension.depth / 2 : plankPosition.z,
        );
        const vectorBase2D = this.projectVectorTo2D(basePosition.x, basePosition.y, basePosition.z);
        this.baseElement.style.top = `${vectorBase2D.y}px`;
        this.baseElement.style.left = `${vectorBase2D.x}px`;
      }
    },
    borderedEdgeID(point1, point2) {
      if (point1[0] === 0 && point2[0] === 0) return 1;
      if (point1[0] === this.x && point2[0] === this.x) return 2;
      if (point1[1] === 0 && point2[1] === 0) return 3;
      if (point1[1] === this.y && point2[1] === this.y) return 4;
      return 0;
    },
    addElementsToDOM() {
      const appDiv = document.getElementById('content-3d');
      const container = document.createElement('div');
      container.classList.add('edges-input-container');

      if (this.plankPoints) {
        this.plankPoints.forEach((point, index) => {
          const edgeElement = this.createSelect();
          edgeElement.value = this.initElement(`point_${index}`);
          this.edgeElements.push(edgeElement);
          container.appendChild(edgeElement);
        });
      } else {
        this.topElement = this.createSelect();
        this.topElement.value = this.initElement('top');
        this.leftElement = this.createSelect();
        this.leftElement.value = this.initElement('left');
        this.rightElement = this.createSelect();
        this.rightElement.value = this.initElement('right');
        this.baseElement = this.createSelect();
        this.baseElement.value = this.initElement('base');
        container.appendChild(this.topElement);
        container.appendChild(this.leftElement);
        container.appendChild(this.rightElement);
        container.appendChild(this.baseElement);
      }

      this.containerElement = container;
      appDiv.insertBefore(container, appDiv.firstChild);
    },
    removeElements() {
      const appDiv = document.getElementById('content-3d');
      appDiv.removeChild(this.containerElement);
    },
    initElement(edgeType) {
      if (edgeType.includes('point')) {
        return this.edges[Number(edgeType.split('_')[1])];
      }

      switch (edgeType) {
        case 'top':
          switch (this.plankType) {
            case 'VP':
              return this.edges[2];
            case 'FP':
              return this.edges[3];
            case 'VDP':
              return this.edges[1];
            default:
              return '0';
          }
        case 'left':
          switch (this.plankType) {
            case 'VP':
              return this.edges[3];
            case 'FP':
              return this.edges[0];
            case 'VDP':
              return this.edges[2];
            default:
              return '0';
          }
        case 'right':
          switch (this.plankType) {
            case 'VP':
              return this.edges[1];
            case 'FP':
              return this.edges[2];
            case 'VDP':
              return this.edges[0];
            default:
              return '0';
          }
        case 'base':
          switch (this.plankType) {
            case 'VP':
              return this.edges[0];
            case 'FP':
              return this.edges[1];
            case 'VDP':
              return this.edges[3];
            default:
              return '0';
          }
        default:
          return '0';
      }
    },
    onChange(edgeType, val) {
      const { value } = val.target;
      const newEdges = [...this.edges];

      if (edgeType.includes('point')) {
        const { plankPoints } = this;
        const edgeIndex = Number(edgeType.split('_')[1]);
        newEdges[edgeIndex] = value;

        if (!this.showAllEdges) {
          const borderNumber = this.borderedEdgeID(plankPoints[edgeIndex], plankPoints[(edgeIndex + 1) % plankPoints.length]);
          plankPoints.forEach((point, index) => {
            const nextIndex = (index + 1) % plankPoints.length;
            if (this.borderedEdgeID(point, plankPoints[nextIndex]) === borderNumber) {
              newEdges[index] = value;
            }
          });
        }
      }

      // edgeType : top, left, right, base
      switch (edgeType) {
        case 'top':
          switch (this.plankType) {
            case 'VP':
              newEdges[2] = value;
              break;
            case 'FP':
              newEdges[3] = value;
              break;
            case 'VDP':
              newEdges[1] = value;
              break;
            default:
              break;
          }
          break;
        case 'left':
          switch (this.plankType) {
            case 'VP':
              newEdges[3] = value;
              break;
            case 'FP':
              newEdges[0] = value;
              break;
            case 'VDP':
              newEdges[2] = value;
              break;
            default:
              break;
          }
          break;
        case 'right':
          switch (this.plankType) {
            case 'VP':
              newEdges[1] = value;
              break;
            case 'FP':
              newEdges[2] = value;
              break;
            case 'VDP':
              newEdges[0] = value;
              break;
            default:
              break;
          }
          break;
        case 'base':
          switch (this.plankType) {
            case 'VP':
              newEdges[0] = value;
              break;
            case 'FP':
              newEdges[1] = value;
              break;
            case 'VDP':
              newEdges[3] = value;
              break;
            default:
              break;
          }
          break;
        default:
          break;
      }
      this.edges = newEdges;
    },
    changeEventHandler(bool) {
      const self = this;
      if (this.plankPoints) {
        if (bool) this.edgeElements.forEach((element, index) => element.addEventListener('change', (val => self.onChange(`point_${index}`, val))));
        else this.edgeElements.forEach((element, index) => element.removeEventListener('change', (val => self.onChange(`point_${index}`, val))));
      } else {
        const handleTopChange = (val => self.onChange('top', val));
        const handleLeftChange = (val => self.onChange('left', val));
        const handleRightChange = (val => self.onChange('right', val));
        const handleBaseChange = (val => self.onChange('base', val));

        if (bool) {
          this.topElement.addEventListener('change', handleTopChange);
          this.leftElement.addEventListener('change', handleLeftChange);
          this.rightElement.addEventListener('change', handleRightChange);
          this.baseElement.addEventListener('change', handleBaseChange);
        } else {
          this.topElement.removeEventListener('change', handleTopChange);
          this.leftElement.removeEventListener('change', handleLeftChange);
          this.rightElement.removeEventListener('change', handleRightChange);
          this.baseElement.removeEventListener('change', handleBaseChange);
        }
      }
    },
  },
  watch: {
    plankPosition() {
      this.updateStyle();
    },
    plankDimension() {
      this.updateStyle();
    },
    plankType() {
      this.updateStyle();
    },
  },
};
</script>
<style>
  .plank-edge-input {
    position: absolute;
  }
</style>
