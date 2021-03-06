<template>
  <div class="vue-tel-input"
       :class="{ disabled: disabled }">
    <div class="dropdown"
         @click="toggleDropdown"
         v-click-outside="clickedOutside"
         :class="{open: open}">
      <span class="selection">
        <div class="iti-flag"
             :class="activeCountry.iso2.toLowerCase()"></div>
        <span class="dropdown-arrow">
          {{ open ? '▲' : '▼' }}
        </span>
      </span>
      <ul v-show="open">
        <li class="dropdown-item"
            v-for="pb in allCountries"
            :key="pb['iso2']"
            @click="choose(pb)">
          <div class="iti-flag"
               :class="pb.iso2.toLowerCase()"></div>
          <strong>{{ pb.name }} </strong>
          <span>+{{ pb.dialCode }}</span>
        </li>
      </ul>
    </div>
    <input v-model="phone"
           :placeholder="placeholder"
           :state="state"
           :formatter="format"
           :disabled="disabled"
           @blur="onBlur"
           @input="onInput">
  </div>
</template>

<style src="./assets/sprite.css"></style>
<style scoped>
:local {
  --border-radius: 2px;
}
.iti-flag {
  margin-right: 5px;
  margin-left: 5px;
}
.dropdown-item .iti-flag {
  display: inline-block;
  margin-right: 5px;
}
.selection {
  cursor: pointer;
  font-size: 0.8em;
  display: flex;
  align-items: center;
}
.vue-tel-input {
  border-radius: 3px;
  display: flex;
  border: 1px solid #bbb;
  text-align: left;
}
.vue-tel-input:focus-within {
  box-shadow: inset 0 1px 1px rgba(0, 0, 0, 0.075),
    0 0 8px rgba(102, 175, 233, 0.6);
  border-color: #66afe9;
}
input {
  border: none;
  border-radius: 0 var(--border-radius) var(--border-radius) 0;
  width: 100%;
  outline: none;
  padding-left: 7px;
}
ul {
  padding: 0;
  margin: 0;
  text-align: left;
  list-style: none;
  max-height: 200px;
  overflow-y: scroll;
  position: absolute;
  top: 33px;
  left: -1px;
  background-color: #fff;
  border: 1px solid #ccc;
  width: 390px;
}
.dropdown {
  display: flex;
  flex-direction: column;
  align-content: center;
  justify-content: center;
  position: relative;
  padding: 7px;
}
.dropdown.open {
  background-color: #f3f3f3;
}
.dropdown:hover {
  background-color: #f3f3f3;
}
.dropdown-arrow {
  transform: scaleY(0.5);
  display: inline-block;
  color: #666;
}
.dropdown-item {
  cursor: pointer;
  padding: 4px 15px;
}
.dropdown-item:hover {
  background-color: #f3f3f3;
}
.dropdown-menu.show {
  max-height: 300px;
  overflow: scroll;
}
.vue-tel-input.disabled .selection,
.vue-tel-input.disabled .dropdown,
.vue-tel-input.disabled input {
  cursor: no-drop;
}
</style>

<script>
import { format, asYouType, isValidNumber } from 'libphonenumber-js';
import allCountries from './assets/all-countries';
import getCountry from './assets/default-country';

export default {
  name: 'vue-tel-input',
  props: {
    value: {
      type: String,
    },
    placeholder: {
      type: String,
      default: 'Enter a phone number',
    },
    disabledFetchingCountry: {
      type: Boolean,
      default: false,
    },
    disabled: {
      type: Boolean,
      default: false,
    },
  },
  mounted() {
    if (!this.disabledFetchingCountry) {
      getCountry().then((res) => {
        this.activeCountry = allCountries.find(country => country.iso2 === res) ||
          allCountries[0];
      });
    }
  },
  created() {
    if (this.value) {
      this.phone = this.value
    }
  },
  data() {
    return {
      phone: '',
      allCountries,
      activeCountry: allCountries[0],
      open: false,
    };
  },
  computed: {
    mode() {
      if (!this.phone) {
        return '';
      }
      if (this.phone[0] === '+') {
        return 'code';
      }
      if (this.phone[0] === '0') {
        return 'prefix';
      }
      return 'normal';
    },
    formattedResult() {
      // Calculate phone number based on mode
      if (!this.mode || !this.allCountries) {
        return '';
      }
      let phone = this.phone;
      if (this.mode === 'code') {
        // If user manually type the country code
        const formatter = new asYouType();// eslint-disable-line
        formatter.input(this.phone);

        // Find inputted country in the countries list
        this.activeCountry = this.allCountries.find(ele =>
          ele.iso2.toUpperCase() === formatter.country) || this.activeCountry;
      } else if (this.mode === 'prefix') {
        // Remove the first '0' if this is a '0' prefix number
        // Ex: 0432421999
        phone = this.phone.slice(1);
      }
      return format(phone, this.activeCountry && this.activeCountry.iso2, 'International');
    },
    state() {
      return isValidNumber(this.formattedResult);
    },
    response() {
      const number = this.formattedResult && this.formattedResult.replace(/ /g, '');
      return {
        number,
        isValid: this.state,
        country: this.activeCountry,
      };
    },
  },
  watch: {
    state(value) {
      if (value) {
        // If mode is 'prefix', keep the number as user typed,
        // Otherwise format it
        this.phone = this.formattedResult;
      }
    },
  },
  methods: {
    choose(country) {
      this.activeCountry = country;
      this.$emit('onInput', this.response);
    },
    onInput() {
      // Emit the input event in case v-model is used in the parent
      this.$emit('input', this.response.number);

      // Emit the response, includes phone, validity and country
      this.$emit('onInput', this.response);
    },
    onBlur() {
      this.$emit('onBlur');
    },
    toggleDropdown: function () {
      if (this.disabled) {
        return;
      }
      this.open = !this.open;
    },
    clickedOutside: function () {
      this.open = false;
    },
  },
  directives: {
    // Click-outside from BosNaufal: https://github.com/BosNaufal/vue-click-outside
    'click-outside': {
      bind: function (el, binding, vNode) {
        // Provided expression must evaluate to a function.
        if (typeof binding.value !== 'function') {
          var compName = vNode.context.name;
          var warn = '[Vue-click-outside:] provided expression ' + binding.expression + ' is not a function, but has to be';
          if (compName) {
            warn += 'Found in component ' + compName;
          }
          console.warn(warn);
        }
        // Define Handler and cache it on the element
        var bubble = binding.modifiers.bubble;
        var handler = function (e) {
          if (bubble || (!el.contains(e.target) && el !== e.target)) {
            binding.value(e)
          }
        };
        el.__vueClickOutside__ = handler;
        // add Event Listeners
        document.addEventListener('click', handler)
      },
      unbind: function (el, binding) {
        // Remove Event Listeners
        document.removeEventListener('click', el.__vueClickOutside__);
        el.__vueClickOutside__ = null
      }
    }
  }
};
</script>
