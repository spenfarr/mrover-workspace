<template>
    <div class="wrap">
        <div class="toggle_drill">
            <Checkbox ref="front_drill" v-bind:name="'Front Drill Active'" v-on:toggle="updateControlMode('front_drill', $event)" />
            <Checkbox ref="back_drill" v-bind:name="'Back Drill Active'" v-on:toggle="updateControlMode('back_drill', $event)" />
        </div>
        <div class="speed_limiter">
            <p>
                Speed Limiter:
                <span>{{ dampenDisplay }}%</span>
            </p>
        </div>
        <div class="mode_info">
            <div v-if="controlMode === 'arm'">
                <p>arm</p>
            </div>
            <template>
                <span v-bind:class="['led', saFrontDrillColor]" />
                <span class="name">Front Drill</span>
                <span v-bind:class="['led', saBackDrillColor]" />
                <span class="name">Back Drill</span>
                <!-- <span v-bind:class="['led', saFrontDroneColor]" />
                <span class="name">Front Drone</span>
                <span v-bind:class="['led', saBackDroneColor]" />
                <span class="name">Back Drone</span> -->
            </template>
        </div>
    </div>
</template>
<script>import Checkbox from './Checkbox.vue'
import { mapGetters, mapMutations } from 'vuex'
import {Toggle, quadratic, deadzone, joystick_math} from '../utils.js'

let interval;
export default {
  data () {
    return {
      saMotor: {
        carriage: 0,
        four_bar: 0,
        front_drill: 0,
        back_drill: 0,
        micro_x: 0,
        micro_y: 0,
        micro_z: 0
        // front_drone: 0,
        // back_drone: 0
      },
      dampen: 0
    }
  },

  computed: {
    saFrontDrillColor: function () {
      return this.saMotor.front_drill !== 0 ? 'lightgreen' : 'red'
    },

    saBackDrillColor: function () {
      return this.saMotor.back_drill !== 0 ? 'lightgreen' : 'red'
    },

    saBackDroneColor: function () {
      return this.saMotor.back_drone !== 0 ? 'lightgreen' : 'red'
    },

    saFrontDroneColor: function () {
      return this.saMotor.front_drone !== 0 ? 'lightgreen' : 'red'
    },

    dampenDisplay: function () {
      return (this.dampen * -50 + 50).toFixed(2)
    },

    ...mapGetters('controls', {
      controlMode: 'controlMode'
    }),
  },

  beforeDestroy: function () {
    window.clearInterval(interval);
  },

  created: function () {

    const JOYSTICK_CONFIG = {
      'forward_back': 1,
      'left_right': 2,
      'dampen': 3,
      'kill': 4,
      'restart': 5,
      'pan': 4,
      'tilt': 5
    }

    const XBOX_CONFIG = {
      'left_js_x': 0,
      'left_js_y': 1,
      'left_trigger': 6,
      'right_trigger': 7,
      'right_js_x': 2,
      'right_js_y': 3,
      'right_bumper': 5,
      'left_bumper': 4,
      'd_pad_up': 12,
      'd_pad_down': 13,
      'd_pad_left': 14,
      'd_pad_right': 15,
      'a': 0,
      'b': 1,
      'x': 2,
      'y': 3
    }
    
    const updateRate = 0.05
    const front_drill_on = new Toggle(false)
    const back_drill_on = new Toggle(false)
    const front_drone_on = new Toggle(false)
    const back_drone_on = new Toggle(false)
    const esc_0_on = new Toggle(false)
    const esc_1_on = new Toggle(false)

    interval = window.setInterval(() => {
      const gamepads = navigator.getGamepads()
      for (let i = 0; i < 4; i++) {
        const gamepad = gamepads[i]
        if (gamepad) {
          if (gamepad.id.includes('Logitech')) {
            const joystickData = {
              'type': 'Joystick',
              'forward_back': gamepad.axes[JOYSTICK_CONFIG['forward_back']],
              'left_right': gamepad.axes[JOYSTICK_CONFIG['left_right']],
              'dampen': gamepad.axes[JOYSTICK_CONFIG['dampen']],
              'kill': gamepad.buttons[JOYSTICK_CONFIG['kill']]['pressed'],
              'restart': gamepad.buttons[JOYSTICK_CONFIG['restart']]['pressed']
            }
            this.dampen = gamepad.axes[JOYSTICK_CONFIG['dampen']]

            if (!this.autonEnabled) {
              this.$parent.publish('/drive_control', joystickData)
            }
          } else if (gamepad.id.includes('Microsoft') || gamepad.id.includes('xinput') || gamepad.id.includes('Xbox')) {
            const xboxData = {
              'type': 'Xbox',
              'left_js_x': gamepad.axes[XBOX_CONFIG['left_js_x']], // actuator 1
              'left_js_y': gamepad.axes[XBOX_CONFIG['left_js_y']], 
              'left_trigger': gamepad.buttons[XBOX_CONFIG['left_trigger']]['value'], //drill motor CCW
              'right_trigger': gamepad.buttons[XBOX_CONFIG['right_trigger']]['value'], //drill motor CW
              'right_js_x': gamepad.axes[XBOX_CONFIG['right_js_x']], //actuator 2
              'right_js_y': gamepad.axes[XBOX_CONFIG['right_js_y']], 
              'right_bumper': gamepad.buttons[XBOX_CONFIG['right_bumper']]['pressed'], //z motor
              'left_bumper': gamepad.buttons[XBOX_CONFIG['left_bumper']]['pressed'], 
              'd_pad_up': gamepad.buttons[XBOX_CONFIG['d_pad_up']]['pressed'], //xy motor
              'd_pad_down': gamepad.buttons[XBOX_CONFIG['d_pad_down']]['pressed'], 
              'd_pad_right': gamepad.buttons[XBOX_CONFIG['d_pad_right']]['pressed'], 
              'd_pad_left': gamepad.buttons[XBOX_CONFIG['d_pad_left']]['pressed'],
              'a': gamepad.buttons[XBOX_CONFIG['a']]['pressed'],
              'b': gamepad.buttons[XBOX_CONFIG['b']]['pressed'],
              'x': gamepad.buttons[XBOX_CONFIG['x']]['pressed'],
              'y': gamepad.buttons[XBOX_CONFIG['y']]['pressed']
              //one of the buttoms to turn on drone motor, servos
            }
            
            // Can only toggle either front or back drill
            let front_drill_input = this.controlMode === 'front_drill' && xboxData['right_bumper']
            let back_drill_input = this.controlMode === 'back_drill' && xboxData['right_bumper']

            const saMotorsData = {
              'type': 'SAMotors',
              'carriage': deadzone(xboxData['right_js_y'], 0.3),
              'four_bar': xboxData['d_pad_left'] - xboxData['d_pad_right'],
              'front_drill': front_drill_input,
              'back_drill': back_drill_input,
              'micro_x': deadzone(xboxData['left_js_x'], 0.3),
              'micro_y': deadzone(xboxData['left_js_y'], 0.3),
              'micro_z': xboxData['right_trigger'] - xboxData['left_trigger']
            }
            this.$parent.publish('/sa_motors', saMotorsData)

            this.$parent.publish('/esc_toggle', {
              'type': 'ESCToggle',
              'id': 'vacuum_1',
              'enable': esc_0_on.new_reading(xboxData['x'] > 0.5)
            })

            this.$parent.publish('/esc_toggle', {
              'type': 'ESCToggle',
              'id': 'vacuum_2',
              'enable': esc_1_on.new_reading(xboxData['b'] > 0.5)
            })
          }
        }
      }

      const talonConfig = {
        'type': 'TalonConfig',
        'enable_arm': false,
        'enable_sa': true,
      }

      this.$parent.publish('/talon_config', talonConfig)
    }, updateRate*1000)

    this.$parent.subscribe('/sa_motors', (msg) => {
      this.saMotor = msg
    })
  },

  methods: {
    updateControlMode: function (mode, checked) {
      if (checked) {
        if (this.controlMode !== ''){
          this.$refs[this.controlMode].toggle()
        }

        this.setControlMode(mode)
      } else {
        this.setControlMode('')
      }
    },

    ...mapMutations('controls', {
      setControlMode: 'setControlMode'
    })
  },

  components: {
    Checkbox
  }
}</script>
<style scoped>

    .wrap {
        display: grid;
        grid-template-areas: "control_buttons speed_limiter mode_info";
        grid-template-columns: 1fr 1fr 1fr;
        align-items: center;
        height: 100%;
        padding: 0px 0px 0px 20px;
    }

    .control_buttons {
        grid-area: control_buttons;
        grid-template-areas: none;
        display: flex;
        padding: 0px 0px 0px 0px;
        height: 100%;
    }

    .button {
        height:20px;
        width:100px;
    }

    .speed_limiter {
        display: flex;
        grid-area: speed_limiter;
        align-items: center;
        justify-content: center;
    }

    .mode_info {
        grid-area: mode_info;
        display: flex;
    }

    .led {
        width: 16px;
        height: 16px;
        border-radius: 8px;
        border: 1px solid;
    }

    .lightgreen {
        background-color: lightgreen;
    }

    .red {
        background-color: red;
    }

    .orange {
        background-color: orange;
    }

    .name {
        margin-left: 5px;
        padding-right: 10px;
    }
</style>
