<template>
  <input
    :placeholder="placeholder"
    @blur="onBlurHandler"
    @input="onInputHandler"
    @focus="onFocusHandler"
    @keydown="onKeyDownHandler"
    ref="numeric"
    type="tel"
    v-model="amount"
    v-if="!readOnly"
  >
  <span
    v-else
    ref="readOnly"
  >{{ amount }}</span>
</template>

<script>
import accounting from 'accounting-js'

export default {
  name: 'VueNumeric',

  props: {
    /**
     * Currency symbol.
     */
    currency: {
      type: String,
      default: '',
      required: false
    },

    /**
     * Maximum value allowed.
     */
    max: {
      type: Number,
      default: Number.MAX_SAFE_INTEGER || 9007199254740991,
      required: false,
    },

    /**
     * Minimum value allowed.
     */
    min: {
      type: Number,
      default: Number.MIN_SAFE_INTEGER || -9007199254740991,
      required: false
    },

    /**
     * Enable/Disable minus value.
     */
    minus: {
      type: Boolean,
      default: false,
      required: false
    },

    /**
     * Input placeholder.
     */
    placeholder: {
      type: String,
      default: '',
      required: false
    },

    /**
     * Value when the input is empty
     */
    emptyValue: {
      validator (val) {
        return val === null || typeof val === 'string' || typeof val === 'number';
      },
      default: '',
      required: false
    },

    /**
     * Number of decimals.
     * Decimals symbol are the opposite of separator symbol.
     */
    precision: {
      type: Number,
      default: 0,
      required: false
    },

    /**
     * Thousand separator type.
     * Separator props accept either . or , (default).
     */
    separator: {
      type: String,
      default: ',',
      required: false
    },

    /**
     * Forced thousand separator.
     * Accepts any string.
     */
    thousandSeparator: {
      default: undefined,
      required: false,
      type: String
    },

    /**
     * Forced decimal separator.
     * Accepts any string.
     */
    decimalSeparator: {
      default: undefined,
      required: false,
      type: String
    },

    /**
      * The output type used for v-model.
      * It can either be String or Number (default).
      */
    outputType: {
      required: false,
        type: String,
        default: 'Number'
    },

    /**
     * v-model value.
     */
    value: {
      validator (val) {
        return val === null || typeof val === 'string' || typeof val === 'number';
      },
      default () {
        return this.emptyValue === null ? null : 0
      },
      required: true
    },

    /**
     * Hide input and show value in text only.
     */
    readOnly: {
      type: Boolean,
      default: false,
      required: false
    },

    /**
     * Class for the span tag when readOnly props is true.
     */
    readOnlyClass: {
      type: String,
      default: '',
      required: false
    },

    /**
     * Position of currency symbol
     * Symbol position props accept either 'suffix' or 'prefix' (default).
     */
    currencySymbolPosition: {
      type: String,
      default: 'prefix',
      required: false
    }
  },

  data: () => ({
    amount: ''
  }),

  computed: {
    /**
     * Number type of formatted value.
     * @return {Number}
     */
    amountNumber () {
      return this.unformat(this.amount)
    },

    /**
     * Number type of value props.
     * @return {Number}
     */
    valueNumber () {
      return this.unformat(this.value)
    },

    /**
     * Define decimal separator based on separator props.
     * @return {String} '.' or ','
     */
    decimalSeparatorSymbol () {
      if (typeof this.decimalSeparator !== 'undefined') return this.decimalSeparator
      if (this.separator === ',') return '.'
      return ','
    },

    /**
     * Define thousand separator based on separator props.
     * @return {String} '.' or ','
     */
    thousandSeparatorSymbol () {
      if (typeof this.thousandSeparator !== 'undefined') return this.thousandSeparator
      if (this.separator === '.') return '.'
      if (this.separator === 'space') return ' '
      return ','
    },

    /**
     * Define format position for currency symbol and value.
     * @return {String} format
     */
    symbolPosition () {
      if (!this.currency) return '%v'
      return this.currencySymbolPosition === 'suffix' ? '%v %s' : '%s %v'
    }
  },

  watch: {
    /**
     * Watch for value change from other input with same v-model.
     * @param {Number} newValue
     */
    valueNumber (newValue) {
      if (this.$refs.numeric !== document.activeElement) {
        this.amount = this.format(newValue)
      }
    },

    /**
     * When readOnly is true, replace the span tag class.
     * @param {Boolean} newValue
     * @param {Boolean} oldValue
     */
    readOnly (newValue, oldValue) {
      if (oldValue === false && newValue === true) {
        this.$nextTick(() => {
          this.$refs.readOnly.className = this.readOnlyClass
        })
      }
    },

    /**
     * Immediately reflect separator changes
     */
    separator () {
      this.process(this.valueNumber)
      this.amount = this.format(this.valueNumber)
    },

    /**
     * Immediately reflect currency changes
     */
    currency () {
      this.process(this.valueNumber)
      this.amount = this.format(this.valueNumber)
    },

    /**
     * Immediately reflect precision changes
     */
    precision () {
      this.process(this.valueNumber)
      this.amount = this.format(this.valueNumber)
    }
  },

  mounted () {
    const vm = this, {format, placeholder, process, valueNumber} = vm;

    // Set default value props when placeholder undefined or placeholder is defined and valueNumber is set.
    if ((placeholder && !vm.isNullOrEmpty(valueNumber)) || !placeholder) {
      process(valueNumber);
      vm.amount = format(valueNumber);

      // In case of delayed props value.
      setTimeout(() => {
        process(valueNumber);
        vm.amount = format(valueNumber);
      }, 500)
    }

    // Set read-only span element's class
    if (vm.readOnly) vm.$refs.readOnly.className = vm.readOnlyClass
  },

  methods: {
    /**
     * Handle blur event.
     * @param {Object} e
     */
    onBlurHandler (e) {
      this.$emit('blur', e)
      this.amount = this.format(this.valueNumber)
    },

    /**
     * Handle focus event.
     * @param {Object} e
     */
    onFocusHandler (e) {
      this.$emit('focus', e)
      if (this.emptyValue !== null && this.valueNumber === 0) {
        this.amount = null
      } else if (this.isNullOrEmpty(this.valueNumber)) {
        this.amount = null;
      } else {
        this.amount = accounting.formatMoney(this.valueNumber, {
          symbol: '',
          format: '%v',
          thousand: '',
          decimal: this.decimalSeparatorSymbol,
          precision: Number(this.precision)
        })
      }
    },
      /**
       * Handle keydown event.
       * @param {Object} e
       */
      onKeyDownHandler (e) {
          this.$emit('keydown', e);
      },

    /**
     * Handle input event.
     */
    onInputHandler () {
      this.process(this.amountNumber)
    },

    isNullOrEmpty (value) {
      return this.emptyValue === null && (value === '' || value === null);
    },

    /**
     * Validate value before update the component.
     * @param {Number} value
     */
    process (value) {
      if (this.isNullOrEmpty(value)) {
        this.update(value)
      } else {
        if (value >= this.max) this.update(this.max)
        if (value <= this.min) this.update(this.min)
        if (value > this.min && value < this.max) this.update(value)
        if (!this.minus && value < 0) this.min >= 0 ? this.update(this.min) : this.update(0)
      }
    },

    /**
     * Update parent component model value.
     * @param {Number} value
     */
    update (value) {
      const fixedValue = accounting.toFixed(value, this.precision)
      const output = this.outputType.toLowerCase() === 'string' ? fixedValue : Number(fixedValue)

      if (this.isNullOrEmpty(value)) {
        this.$emit('input', null)
      } else {
        this.$emit('input', output)
      }
    },

    /**
     * Format value using symbol and separator.
     * @param {Number} value
     * @return {String}
     */
    format (value) {
      if (value === null) {
        return null;
      }

      return accounting.formatMoney(value, {
        symbol: this.currency,
        format: this.symbolPosition,
        precision: Number(this.precision),
        decimal: this.decimalSeparatorSymbol,
        thousand: this.thousandSeparatorSymbol
      })
    },

    /**
     * Remove symbol and separator.
     * @param {Number} value
     * @return {Number}
     */
    unformat (value) {
      const toUnformat = typeof value === 'string' && value === '' ? this.emptyValue : value

      if (toUnformat === null) {
        return null;
      }

      return accounting.unformat(toUnformat, this.decimalSeparatorSymbol)
    }
  }
}
</script>
