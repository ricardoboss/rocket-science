<template>
    <v-stage id="main_stage" :config="configKonva" @resize="resizeCanvas">
      <v-layer ref="path_layer">
        <v-line :config="{points: path_marks, stroke: 'white', strokeWidth: 1, opacity: 0.05}"></v-line>
      </v-layer>
      <v-layer>
        <v-circle :config="planetConfig"></v-circle>
      </v-layer>
      <v-layer>
        <v-arrow :config="gravityVectorConfig"></v-arrow>
        <v-arrow :config="rocketVectorConfig"></v-arrow>
        <v-arrow :config="rocketConfig"></v-arrow>
      </v-layer>
      <v-layer>
        <v-group>
          <v-text :config="{x: 10, y: 10, fill: 'white', text: `pos: x: ${this.rocketPosition.x.toFixed(2)} y: ${this.rocketPosition.y.toFixed(2)} phi: ${this.rocketPosition.phi.toFixed(2)}`}"></v-text>
          <v-text :config="{x: 10, y: 25, fill: 'white', text: `vel: x: ${this.rocketVelocity.x.toFixed(2)} y: ${this.rocketVelocity.y.toFixed(2)} phi: ${this.rocketVelocity.phi.toFixed(2)}`}"></v-text>
          <v-text :config="{x: 10, y: 40, fill: 'white', text: `distance: ${(this.distanceBetweenCenters - this.planetRadius - this.rocketRadius / 2).toFixed(2)}`}"></v-text>
          <v-text :config="{x: 10, y: 55, fill: 'white', text: `boosting: ${this.boost ? 'yes' : 'no'}`}"></v-text>
          <v-text :config="{x: 10, y: 70, fill: 'white', text: `tracking trajectory: ${this.track_path === true ? 'yes' : (this.track_path === false ? 'yes (reversed)' : 'no')}`}"></v-text>
        </v-group>
        <v-group>
          <v-text :config="{x: 10, y: this.configKonva.height - 20, fill: 'white', text: `Space - toggle tracking direction`}"></v-text>
          <v-text :config="{x: 10, y: this.configKonva.height - 35, fill: 'white', text: `F - toggle tracking trajectory`}"></v-text>
          <v-text :config="{x: 10, y: this.configKonva.height - 50, fill: 'white', text: `S - slow down spin`}"></v-text>
          <v-text :config="{x: 10, y: this.configKonva.height - 65, fill: 'white', text: `A/D - spin left/right`}"></v-text>
          <v-text :config="{x: 10, y: this.configKonva.height - 80, fill: 'white', text: `W - boost`}"></v-text>
        </v-group>
        <v-group>
          <v-circle :config="{x: this.configKonva.width / 2, y: this.configKonva.height - 50, radius: 50, fill: 'black', stroke: 'white', strokeWidth: 1}"></v-circle>
          <v-arrow :config="previewPlanetVectorConfig"></v-arrow>
          <v-arrow :config="previewRocketVectorConfig"></v-arrow>
          <v-arrow :config="previewRocketConfig"></v-arrow>
        </v-group>
      </v-layer>
    </v-stage>
</template>

<script>
const degToRad = Math.PI / 180;
const bigG = 6.6742;

export const planetRadius = 100;
const planetMass = 10000;
export const rocketRadius = 5;
const rocketMass = 0.1;
const launchFactor = 5;
const thrustFactor = 0.02;
const surfaceTolerance = 0.1;
const frictionFactor = 1 - 0.000001;

const pathMaxPoints = 10000;

export default {
  name: "Game",

  created() {
    window.addEventListener("resize", this.resizeCanvas);
    window.addEventListener("keydown", this.keydown);
    window.addEventListener("keyup", this.keyup);
  },

  destroyed() {
    window.removeEventListener("resize", this.resizeCanvas);
    window.removeEventListener("keydown", this.keydown);
    window.removeEventListener("keyup", this.keyup);

    clearInterval(this.gameTickInterval);
  },

  mounted() {
    this.resizeCanvas();

    this.gameTickInterval = setInterval(() => this.gameTick(), 20)
  },

  data() {
    return {
      gameTickInterval: -1,

      resizeTrigger: false,

      planetRotation: 0,
      planetRadius,
      rocketRadius,

      rocketLanded: true,
      rocketLandingOffset: 0,
      rocketVelocity: {
        x: 0,
        y: 0,
        phi: 0,
      },
      rocketPosition: {
        x: 0,
        y: 0,
        phi: 0
      },

      distanceBetweenCenters: 0,
      planetToRocket: {
        x: 0,
        y: 0
      },
      gravity: {
        x: 0,
        y: 0
      },

      boost: false,
      rotate: null,
      track_path: null,

      path_marks: []
    };
  },

  methods: {
    gameTick() {
      this.planetRotation = (this.planetRotation + 0.1) % 360;

      this.distanceBetweenCenters = Math.sqrt(Math.pow(this.planetConfig.x - this.rocketPosition.x, 2) + Math.pow(this.planetConfig.y - this.rocketPosition.y, 2));
      this.planetToRocket.x = (this.rocketPosition.x - this.planetConfig.x) / this.distanceBetweenCenters;
      this.planetToRocket.y = (this.rocketPosition.y - this.planetConfig.y) / this.distanceBetweenCenters;

      if (this.distanceBetweenCenters < rocketRadius / 2 + planetRadius - surfaceTolerance) {
        this.rocketLanded = true;

        this.rocketLandingOffset = this.planetRotation;
      }

      if (this.rocketLanded) {
        this.rocketPosition.x = this.planetConfig.x + Math.cos((this.planetRotation + this.rocketLandingOffset) * degToRad) * (planetRadius + rocketRadius / 2);
        this.rocketPosition.y = this.planetConfig.y + Math.sin((this.planetRotation + this.rocketLandingOffset) * degToRad) * (planetRadius + rocketRadius / 2);
        this.rocketPosition.phi = (this.planetRotation + this.rocketLandingOffset) % 360;

        this.rocketVelocity.x = 0;
        this.rocketVelocity.y = 0;
        this.rocketVelocity.phi = 0;
      } else {
        if (this.boost)
        {
          let thrust = {
            x: Math.cos(this.rocketPosition.phi * degToRad) * thrustFactor,
            y: Math.sin(this.rocketPosition.phi * degToRad) * thrustFactor
          };

          this.rocketVelocity.x += thrust.x;
          this.rocketVelocity.y += thrust.y;
        }

        if (this.rotate > 0)
          this.rocketVelocity.phi += 0.05;
        else if (this.rotate < 0)
          this.rocketVelocity.phi -= 0.05;
        else if (this.rotate === 0)
          this.rocketVelocity.phi *= 0.99;

        let gravitationalForce = bigG * (planetMass * rocketMass) / Math.pow(this.distanceBetweenCenters, 2);
        this.gravity.x = gravitationalForce * this.planetToRocket.x;
        this.gravity.y = gravitationalForce * this.planetToRocket.y;

        this.rocketVelocity.x -= this.gravity.x;
        this.rocketVelocity.y -= this.gravity.y;
        this.rocketVelocity.phi %= 360;

        this.rocketVelocity.x *= frictionFactor;
        this.rocketVelocity.y *= frictionFactor;

        this.rocketPosition.x += this.rocketVelocity.x;
        this.rocketPosition.y += this.rocketVelocity.y;

        if (this.track_path === true) {
          this.rocketPosition.phi = Math.atan2(this.rocketVelocity.y, this.rocketVelocity.x) / degToRad;
        } else if (this.track_path === false) {
          this.rocketPosition.phi = Math.atan2(this.rocketVelocity.y, this.rocketVelocity.x) / degToRad - 180;
        } else {
          this.rocketPosition.phi += this.rocketVelocity.phi;
        }
        this.rocketPosition.phi %= 360;

        this.path_marks.push(this.rocketPosition.x);
        this.path_marks.push(this.rocketPosition.y);
        if (this.path_marks.length > pathMaxPoints)
          this.path_marks.splice(0, 2);

        this.$refs.path_layer.getNode().draw();
      }
    },

    resizeCanvas() {
      // toggle the boolean to trigger computed values to re-compute
      this.resizeTrigger = !this.resizeTrigger;
    },

    keydown(e) {
      switch (e.keyCode)
      {
        case 87: // w
          this.boost = true;

          if (this.rocketLanded) {
            this.rocketLanded = false;
            this.track_path = true;

            // orient tangential to surface
            this.rocketPosition.phi = this.planetRotation + 90;

            this.rocketVelocity.x = (this.planetToRocket.x + Math.cos(this.rocketPosition.phi * degToRad)) * launchFactor;
            this.rocketVelocity.y = (this.planetToRocket.y + Math.sin(this.rocketPosition.phi * degToRad)) * launchFactor;
          }
          break;
        case 65: // a
          if (!this.rocketLanded)
            this.rotate = -1;
          break;
        case 68: // d
          if (!this.rocketLanded)
            this.rotate = 1;
          break;
        case 70: // f
          this.rocketVelocity.phi = 0;
          if (this.track_path === null)
            this.track_path = true; // track trajectory
          else
            this.track_path = null; // stop tracking
          break;
        case 32: // space
          this.track_path = !this.track_path; // reverse trajectory tracking
          break;
        case 83: // s
          if (!this.rocketLanded)
            this.rotate = 0;
          break;
        default:
          console.log(e.keyCode);
          break;
      }
    },

    keyup(e) {
      switch (e.keyCode) {
        case 87: // w
          this.boost = false;
          break;
        case 65: // a
        case 68: // d
        case 83: // s
          if (!this.rocketLanded)
            this.rotate = null;
          break;
      }
    }
  },

  computed: {
    configKonva() {
      // noinspection BadExpressionStatementJS
      this.resizeTrigger;

      return {
        width: window.innerWidth,
        height: window.innerHeight,
      };
    },

    planetConfig() {
      return {
        x: this.configKonva.width / 2,
        y: this.configKonva.height / 2,
        fill: 'blue',
        rotation: this.planetRotation,
        radius: planetRadius,
      };
    },

    rocketConfig() {
      return {
        points: [1,1],
        x: this.rocketPosition.x,
        y: this.rocketPosition.y,
        fill: 'green',
        rotation: this.rocketPosition.phi,
        pointerWidth: 4 * rocketRadius / 5,
        pointerLength: rocketRadius,
        stroke: 'green'
      }
    },

    previewRocketConfig() {
      return {
        points: [0,0],
        x: this.configKonva.width / 2,
        y: this.configKonva.height - 50,
        fill: 'green',
        rotation: this.rocketPosition.phi,
        pointerWidth: 4 * rocketRadius / 5,
        pointerLength: rocketRadius,
        stroke: 'green'
      }
    },

    rocketVectorConfig() {
      return {
        points: [
          this.rocketVelocity.x * 2, this.rocketVelocity.y * 2,
          this.rocketVelocity.x * 5, this.rocketVelocity.y * 5,
        ],
        x: this.rocketPosition.x,
        y: this.rocketPosition.y,
        fill: 'red',
        stroke: 'red',
        pointerWidth: 3,
        pointerLength: 3,
        visible: this.rocketVelocity.x !== 0 && this.rocketVelocity.y !== 0,
      }
    },

    previewRocketVectorConfig() {
      return {
        points: [
          this.rocketVelocity.x * 5, this.rocketVelocity.y * 5,
          this.rocketVelocity.x * 10, this.rocketVelocity.y * 10,
        ],
        x: this.configKonva.width / 2,
        y: this.configKonva.height - 50,
        fill: 'red',
        stroke: 'red',
        pointerWidth: 3,
        pointerLength: 3,
        visible: this.rocketVelocity.x !== 0 && this.rocketVelocity.y !== 0,
      }
    },

    gravityVectorConfig() {
      return {
        points: [
          -this.planetToRocket.x * 5, -this.planetToRocket.y * 5,
          -this.planetToRocket.x * 10, -this.planetToRocket.y * 10
        ],
        x: this.rocketPosition.x,
        y: this.rocketPosition.y,
        fill: 'blue',
        stroke: 'blue',
        pointerWidth: 3,
        pointerLength: 3,
        visible: this.planetToRocket.x !== 0 && this.planetToRocket.y !== 0,
      }
    },

    previewPlanetVectorConfig()
    {
      return {
        points: [
          0,0,
          -this.planetToRocket.x * 50, -this.planetToRocket.y * 50
        ],
        x: this.configKonva.width / 2,
        y: this.configKonva.height - 50,
        fill: 'blue',
        stroke: 'blue',
        strokeWidth: 0,
        pointerWidth: 6,
        pointerLength: 6,
      }
    },
  }
}
</script>

<style scoped lang="scss">
#main_stage {
  width: 100% !important;
  height: 100% !important;
  background-color: black;
}
</style>
