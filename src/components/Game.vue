<template>
    <v-stage id="main_stage" ref="main_stage" :config="configKonva" @resize="resizeCanvas">
      <v-layer>
        <v-ellipse :config="orbitConfig"></v-ellipse>
        <v-circle :config="planetConfig"></v-circle>
      </v-layer>
      <v-layer ref="path_layer">
        <v-line :config="{points: path_marks, stroke: 'white', strokeWidth: 1, opacity: 0.02}"></v-line>
      </v-layer>
      <v-layer>
        <v-arrow :config="rocketConfig"></v-arrow>
        <v-arrow :config="rocketVectorConfig"></v-arrow>
      </v-layer>
      <v-layer>
        <v-text :config="{x: 10, y: 10, fill: 'white', text: `pos: x: ${this.rocketPosition.x.toFixed(2)} y: ${this.rocketPosition.y.toFixed(2)} phi: ${this.rocketPosition.phi.toFixed(2)}`}"></v-text>
        <v-text :config="{x: 10, y: 25, fill: 'white', text: `vel: x: ${this.rocketVelocity.x.toFixed(2)} y: ${this.rocketVelocity.y.toFixed(2)} phi: ${this.rocketVelocity.phi.toFixed(2)}`}"></v-text>
        <v-text :config="{x: 10, y: 40, fill: 'white', text: `distance: ${this.distanceBetweenCenters.toFixed(2)}`}"></v-text>
        <v-text :config="{x: 10, y: 55, fill: 'white', text: `boosting: ${this.boost ? 'yes' : 'no'}`}"></v-text>
        <v-text :config="{x: 10, y: 70, fill: 'white', text: `tracking trajectory: ${this.track_path === true ? 'yes' : (this.track_path === false ? 'yes (reversed)' : 'no')}`}"></v-text>
      </v-layer>
      <v-layer>
        <v-text :config="{x: 10, y: this.configKonva.height - 20, fill: 'white', text: `Space - toggle tracking direction`}"></v-text>
        <v-text :config="{x: 10, y: this.configKonva.height - 35, fill: 'white', text: `F - toggle tracking trajectory`}"></v-text>
        <v-text :config="{x: 10, y: this.configKonva.height - 50, fill: 'white', text: `S - slow down spin`}"></v-text>
        <v-text :config="{x: 10, y: this.configKonva.height - 65, fill: 'white', text: `A/D - spin left/right`}"></v-text>
        <v-text :config="{x: 10, y: this.configKonva.height - 80, fill: 'white', text: `W - boost`}"></v-text>
      </v-layer>
      <v-layer>
        <v-circle :config="{x: this.configKonva.width / 2, y: this.configKonva.height - 50, radius: 50, fill: 'black', stroke: 'white', strokeWidth: 1}"></v-circle>
        <v-arrow :config="previewRocketConfig"></v-arrow>
        <v-arrow :config="previewRocketVectorConfig"></v-arrow>
      </v-layer>
    </v-stage>
</template>

<script>
const degToRad = Math.PI / 180;
const bigG = 6.6742;

const planetRadius = 75;
const planetMass = 10;
const rocketRadius = 5;
const rocketMass = 0.1;
const launchSpeed = 6;

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

    this.gameTickInterval = setInterval(() => this.gameTick(), 10)
  },

  data() {
    return {
      gameTickInterval: -1,

      resizeTrigger: false,

      planetRotation: 0,

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

      boost: false,
      rotate: null,
      track_path: null,

      orbit: {
        center: {
          x: 0,
          y: 0
        },
        radius: {
          x: 0,
          y: 0
        }
      },

      path_marks: []
    };
  },

  methods: {
    gameTick() {
      this.planetRotation = (this.planetRotation + 0.1) % 360;

      this.distanceBetweenCenters = Math.sqrt(Math.pow(this.planetConfig.x - this.rocketPosition.x, 2) + Math.pow(this.planetConfig.y - this.rocketPosition.y, 2));
      this.planetToRocket.x = (this.rocketPosition.x - this.planetConfig.x) / this.distanceBetweenCenters;
      this.planetToRocket.y = (this.rocketPosition.y - this.planetConfig.y) / this.distanceBetweenCenters;

      let distanceToSurface = this.distanceBetweenCenters - (rocketRadius / 2 + planetRadius / 2);
      if (distanceToSurface < rocketRadius) {
        this.rocketLanded = true;

        this.rocketLandingOffset = this.planetRotation;
      }

      if (this.rocketLanded) {
        this.rocketPosition.x = this.planetConfig.x + Math.cos((this.planetRotation + this.rocketLandingOffset) * degToRad) * (planetRadius + rocketRadius / 2);
        this.rocketPosition.y = this.planetConfig.y + Math.sin((this.planetRotation + this.rocketLandingOffset) * degToRad) * (planetRadius + rocketRadius / 2);
        this.rocketPosition.phi = this.planetRotation + this.rocketLandingOffset;

        this.orbit.radius.x = planetRadius + rocketRadius / 2;
        this.orbit.radius.y = planetRadius + rocketRadius / 2;

        this.rocketVelocity.x = 0;
        this.rocketVelocity.y = 0;
        this.rocketVelocity.phi = 0;
      } else {
        if (this.boost)
        {
          let thrust_factor = 0.003;

          let thrust = {
            x: Math.cos(this.rocketPosition.phi * degToRad) * thrust_factor,
            y: Math.sin(this.rocketPosition.phi * degToRad) * thrust_factor
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

        let gravitationalForce = bigG * (planetMass * rocketMass) / distanceToSurface;
        let gravity = {
          x: gravitationalForce * this.planetToRocket.x,
          y: gravitationalForce * this.planetToRocket.y
        }

        this.rocketVelocity.x -= gravity.x;
        this.rocketVelocity.y -= gravity.y;
        this.rocketVelocity.phi %= 360;

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

        this.orbit.radius.x = this.distanceBetweenCenters;
        this.orbit.radius.y = this.distanceBetweenCenters;
      }

      this.path_marks.push(this.rocketPosition.x);
      this.path_marks.push(this.rocketPosition.y);
      this.$refs.path_layer.getNode().draw();
    },

    resizeCanvas() {
      // toggle the boolean to trigger computed values to re-compute
      this.resizeTrigger = !this.resizeTrigger;

      this.orbit.center.x = this.planetConfig.x;
      this.orbit.center.y = this.planetConfig.y;
    },

    keydown(e) {
      switch (e.keyCode)
      {
        case 87: // w
          if (this.rocketLanded) {
            this.rocketLanded = false;

            this.rocketVelocity.x = this.planetToRocket.x * launchSpeed;
            this.rocketVelocity.y = this.planetToRocket.y * launchSpeed;
            this.rocketPosition.phi = this.planetRotation + 90;
          } else {
            this.boost = true;
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
          this.rocketVelocity.x * 5, this.rocketVelocity.y * 5,
          this.rocketVelocity.x * 10, this.rocketVelocity.y * 10,
        ],
        x: this.rocketPosition.x,
        y: this.rocketPosition.y,
        fill: 'red',
        stroke: 'red',
        pointerWidth: 3,
        pointerLength: 3,
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
      }
    },

    orbitConfig() {
      return {
        radius: {
          x: this.orbit.radius.x,
          y: this.orbit.radius.y,
        },
        x: this.orbit.center.x,
        y: this.orbit.center.y,
        stroke: 'gray',
        strokeWidth: 0.25,
        fillEnabled: false,
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
