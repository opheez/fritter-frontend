<!-- Reusable component representing a form in a block style -->
<!-- This is just an example; feel free to define any reusable components you want! -->

<template>
  <form @submit.prevent="submit">
    <h3>{{ title }}</h3>
    <article
      v-if="fields.length"
    >
      <div
        v-for="field in fields"
        :key="field.id"
      >
        <label :for="field.id">
          {{ field.label }}{{field.required ? '*':''}}
          <div v-if="field.tooltip" class="tooltip">?
            <span class="tooltiptext">{{field.tooltip}}</span>
          </div>
          :
        </label>
        <textarea
          v-if="field.id === 'content'"
          :name="field.id"
          :value="field.value"
          @input="field.value = $event.target.value"
          placeholder="Write your freet here..."
        />
        <input
          v-else-if="field.id === 'name'"
          :name="field.id"
          :value="field.value"
          @input="field.value = $event.target.value"
          />
          <div
          v-else-if="field.id === 'public'"
          >
          <label class="switch">
            <input
              type="checkbox"
              v-model="field.value"
              @change="field.value = $event.target.checked"
            >
            <span class="slider round"></span>
          </label>
        </div>
        <div v-else-if="field.id === 'users'">
          <textarea
            :name="field.id"
            :value="field.text"
            @keyup.enter="search(field.id)"
            @input="field.text = $event.target.value"
          />
          <div class="results">
            <article class="user" v-for="user in searchedUsers">
              <p class="name">
                {{ user.username }}
              </p>
              <div class="actions">
                <button type="button" @click="addUser(user)">
                  + Add
                </button>
              </div>
            </article>
          </div>
          <h4> Selected Users:</h4>
          <div class="results">
            <p v-if="Object.keys(selectedUsers).length == 0">No users selected. Add a user by searching above.</p>
            <article 
              v-else
              class="user"
              v-for="user in selectedUsers"
            >
              <p class="name">
                {{ user.username }}
              </p>
              <div class="actions">
                <button type="button" @click="removeUser(user)">
                  🗑️ Remove
                </button>
              </div>
            </article>
          </div>
        </div>
        <input
          v-else-if="field.id === 'tagLabels'"
          :name="field.id"
          :value="field.value"
          @input="field.value = $event.target.value"
        />
        <select
          v-else-if="field.id === 'intent' && field.multiple === true"
          :name="field.id"
          v-model="field.value"
          multiple
        >
          <option 
              v-for="option in field.options"
              :value="option"
          >
            {{option}}
          </option>
        </select>
        <select
          v-else-if="field.id === 'intent' && field.multiple === true"
          :name="field.id"
          v-model="field.value"
          multiple
        >
          <option 
              v-for="option in field.options"
              :value="option"
          >
            {{option}}
          </option>
        </select>
        <select 
          v-else-if="field.id === 'intent'"
          :name="field.id"
          v-model="field.value"
          @input="field.value = $event.target.value"
        >
          <option 
              v-for="option in field.options"
              :value="option"
          >
            {{option}}
          </option>
        </select>
        <input
          v-else
          :type="field.id === 'password' ? 'password' : 'text'"
          :name="field.id"
          :value="field.value"
          @input="field.value = $event.target.value"
        >
      </div>
    </article>
    <article v-else>
      <p>{{ content }}</p>
    </article>
    <button
      type="submit"
    >
      {{ title }}
    </button>
    <section class="alerts">
      <article
        v-for="(status, alert, index) in alerts"
        :key="index"
        :class="status"
      >
        <p>{{ alert }}</p>
      </article>
    </section>
  </form>
</template>

<script>
import Vue from 'vue';

export default {
  name: 'BlockForm',
  data() {
    /**
     * Options for submitting this form.
     */
    return {
      url: '', // Url to submit form to
      method: 'GET', // Form request method
      hasBody: false, // Whether or not form request has a body
      setUsername: false, // Whether or not stored username should be updated after form submission
      refreshFreets: false, // Whether or not stored freets should be updated after form submission
      alerts: {}, // Displays success/error messages encountered during form submission
      callback: null, // Function to run after successful form submission
      searchedUsers: {},
      selectedUsers: {},
    };
  },
  methods: {
    async search(id){
      for (let field of this.fields) {
        if (field.id == id) {
          // submit 
          try {
            const r = await fetch(field.url + `${field.text}`);
            const res = await r.json();
            if (!r.ok) {
              throw new Error(res.error);
            } 
            if (res.users.length == 0) {
              throw new Error('No users found');
            }
            for (let user of res.users){
              Vue.set(this.searchedUsers, user._id, user);
            }
          } catch (e) {
            this.$set(this.alerts, e, 'error');
            setTimeout(() => this.$delete(this.alerts, e), 3000);
          }
          field.text = field.defaultVal;
        }
      }
    },
    removeUser(user){
      Vue.delete(this.selectedUsers, user._id);
      Vue.set(this.searchedUsers, user._id, user);
    },
    addUser(user){
      Vue.delete(this.searchedUsers, user._id);
      Vue.set(this.selectedUsers, user._id, user);
    },
    async submit() {
      /**
        * Submits a form with the specified options from data().
        */
      const options = {
        method: this.method,
        headers: {'Content-Type': 'application/json'},
        credentials: 'same-origin' // Sends express-session credentials with request
      };
      if (this.hasBody) {
        options.body = JSON.stringify(Object.fromEntries(
          this.fields.map(field => {
            const {id, value} = field;
            field.value = '';
            return [id, value];
          })
        ));
      }

      try {
        const r = await fetch(this.url, options);
        if (!r.ok) {
          // If response is not okay, we throw an error and enter the catch block
          const res = await r.json();
          throw new Error(res.error);
        }

        if (this.setUsername) {
          const text = await r.text();
          const res = text ? JSON.parse(text) : {user: null};
          this.$store.commit('setUsername', res.user ? res.user.username : null);
          this.$store.commit('refreshCustomFilters');
        }

        if (this.refreshFreets) {
          this.$store.commit('refreshFreets');
        }

        if (this.callback) {
          this.callback();
        }
      } catch (e) {
        this.$set(this.alerts, e, 'error');
        setTimeout(() => this.$delete(this.alerts, e), 3000);
      }
    }
  }
};
</script>

<style scoped>
form {
  border: 1px solid #111;
  padding: 0.5rem;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  margin-bottom: 14px;
  position: relative;
}

article > div {
  display: flex;
  flex-direction: column;
}

form > article p {
  margin: 0;
}

form h3,
form > * {
  margin: 0.3em 0;
}

form h3 {
  margin-top: 0;
}

textarea {
   font-family: inherit;
   font-size: inherit;
}
</style>
