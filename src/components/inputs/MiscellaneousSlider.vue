<style scoped>
._fan-slider-subheader {
    height: auto;
}
</style>

<template>
    <v-container class="px-0 py-2">
        <v-row>
            <v-col :class="pwm ? 'pb-1' : 'pb-3'">
                <v-subheader class="_fan-slider-subheader">
                    <v-btn v-if="lockSliders && isTouchDevice && pwm" plain small icon @click="isLocked = !isLocked">
                        <v-icon small :color="isLocked ? 'red' : ''">
                            {{ isLocked ? 'mdi-lock-outline' : 'mdi-lock-open-variant-outline' }}
                        </v-icon>
                    </v-btn>
                    <v-icon
                        v-if="type !== 'output_pin'"
                        small
                        :class="'mr-2 ' + (value >= off_below && value > 0 ? 'icon-rotate' : '')">
                        mdi-fan
                    </v-icon>
                    <span>{{ convertName(name) }}</span>
                    <v-spacer></v-spacer>
                    <small v-if="rpm || rpm === 0" :class="'mr-3 ' + (rpm === 0 && value > 0 ? 'red--text' : '')">
                        {{ Math.round(rpm) }} RPM
                    </small>
                    <span v-if="!controllable || (controllable && pwm)" class="font-weight-bold">
                        {{ Math.round(parseFloat(value) * 100) }} %
                    </span>
                    <v-icon v-if="controllable && !pwm" @click="switchOutputPin">
                        {{ value ? 'mdi-toggle-switch' : 'mdi-toggle-switch-off-outline' }}
                    </v-icon>
                </v-subheader>
                <v-card-text v-if="controllable && pwm" class="py-0">
                    <v-slider
                        v-model="value"
                        v-touch="{ start: resetLockTimer }"
                        :disabled="isLocked"
                        :min="0.0"
                        :max="1.0"
                        :step="0.01"
                        :color="value < off_below && value > 0 ? 'red' : undefined"
                        hide-details
                        @change="changeSlider">
                        <template #prepend>
                            <v-icon :disabled="isLocked" @click="decrement">mdi-minus</v-icon>
                        </template>

                        <template #append>
                            <v-icon :disabled="isLocked" @click="increment">mdi-plus</v-icon>
                        </template>
                    </v-slider>
                </v-card-text>
            </v-col>
        </v-row>
    </v-container>
</template>

<script lang="ts">
import { convertName } from '@/plugins/helpers'
import { Component, Mixins, Prop, Watch } from 'vue-property-decorator'
import BaseMixin from '@/components/mixins/base'
import { Debounce } from 'vue-debounce-decorator'

@Component
export default class MiscellaneousSlider extends Mixins(BaseMixin) {
    convertName = convertName
    private timeout: number | undefined
    private isLocked = false
    private min = 0
    private value = 0

    @Prop({ type: Number, required: true }) declare target: number
    @Prop({ type: Number, default: 1 }) declare max: number
    @Prop({ type: String, default: '' }) declare name: string
    @Prop({ type: String, default: '' }) declare type: string
    @Prop({ type: Boolean, default: false }) declare controllable: boolean
    @Prop({ type: Boolean, default: false }) declare pwm: boolean
    @Prop({ type: [Number, Boolean], default: false }) declare rpm: number | boolean
    @Prop({ type: Number, default: 1 }) declare multi: number
    @Prop({ type: Number, default: 0 }) declare off_below: number

    @Watch('lockSliders', { immediate: true })
    lockSlidersChanged() {
        this.isLocked = this.lockSliders && this.isTouchDevice
    }

    startLockTimer() {
        let t = this.lockSlidersDelay
        if (!this.isTouchDevice || !this.lockSliders || t <= 0) return
        this.timeout = setTimeout(() => (this.isLocked = true), t * 1000)
    }

    resetLockTimer() {
        clearTimeout(this.timeout)
    }

    get lockSliders() {
        return this.$store.state.gui.uiSettings.lockSlidersOnTouchDevices
    }

    get lockSlidersDelay() {
        return this.$store.state.gui.uiSettings.lockSlidersDelay
    }

    @Debounce(500)
    changeSlider() {
        this.sendCmd()
    }

    sendCmd() {
        let gcode = ''

        if (this.value < this.min) this.value = 0
        if (this.target === this.value) return

        const l_value = this.value * this.multi

        if (this.type === 'fan') gcode = 'M106 S' + l_value.toFixed(0)
        if (this.type === 'fan_generic') gcode = 'SET_FAN_SPEED FAN=' + this.name + ' SPEED=' + l_value
        if (this.type === 'output_pin') gcode = 'SET_PIN PIN=' + this.name + ' VALUE=' + l_value.toFixed(2)

        if (gcode !== '') {
            this.$store.dispatch('server/addEvent', { message: gcode, type: 'command' })
            this.$socket.emit('printer.gcode.script', { script: gcode })
        }

        this.startLockTimer()
    }

    switchOutputPin() {
        this.value = this.value ? 0 : 1
        const gcode = 'SET_PIN PIN=' + this.name + ' VALUE=' + (this.value * this.multi).toFixed(2)
        this.$store.dispatch('server/addEvent', { message: gcode, type: 'command' })
        this.$socket.emit('printer.gcode.script', { script: gcode })
    }

    decrement() {
        this.value = this.value > 0 ? Math.round((this.value - 0.01) * 100) / 100 : 0
        this.sendCmd()
    }

    increment() {
        this.value = this.value < 1.0 ? Math.round((this.value + 0.01) * 100) / 100 : 1.0
        if (this.value < this.off_below) this.value = this.off_below
        this.sendCmd()
    }

    mounted() {
        this.value = this.target
    }

    @Watch('target')
    targetChanged(newVal: number) {
        this.value = newVal / this.max
    }
}
</script>
