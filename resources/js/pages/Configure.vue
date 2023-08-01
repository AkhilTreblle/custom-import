<template>
    <div>
      <Head>
        <title>Configure Import</title>
      </Head>
  
      <heading class="mb-6">CSV Import - Configure</heading>
  
      <card class="p-8 space-y-4 mb-8">
        <p>
          We were able to discover <b>{{ headings.length }}</b> column(s) and <b>{{ total_rows }}</b>
          row(s) in your data.
        </p>
  
        <p>
          Here's a sample of the data:
        </p>
  
        <hr>
  
        <div class="overflow-scroll">
          <table cellpadding="10">
            <thead class="border-b">
              <tr>
                <th v-for="heading in headings"><span class="font-bold">{{ heading }}</span></th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="row in rows">
                <td v-for="col in row">
                  <code>
                    {{ col }}
                    <i v-if="!col"></i>
                  </code>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </card>
  
      <card class="p-8 space-y-4">
        <template v-if="resource">
          <table cellpadding="10">
            <thead class="border-b">
              <tr>
                <th>Field</th>
                <th>Value</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="field in fields[resource]">
                <td class="pr-2"  :class="{ 'hidden': field.attribute !== 'source' }">
                  <span class="font-bold">{{ field.name }}</span><br>
                  <small class="text-grey-300">{{ field.attribute }}</small>
                </td>
                <td class="space-y-2">
                    <SelectControl @change="(value) => updateMapping(field.attribute, value)" :selected="getSelectedValue(field)"  :class="{ 'hidden': field.attribute !== 'source' }">
                      <option value="" disabled selected>Choose a source</option> <!-- Placeholder option -->
                      <option value="custom" v-if="field.attribute === 'source'">Text</option>
                      <option value="" v-else-if="field.rules.includes('required')">- This field is required -</option>
                      <option value="" v-else>- Leave field empty -</option>
                      <optgroup label="File columns" v-if="field.attribute !== 'source'">
                      <option v-for="heading in headings" :value="heading">{{ heading }}</option>
                      </optgroup>
                    </SelectControl>
  
                  <input v-model="values[field.attribute]" v-if="mappings[field.attribute] === 'custom'" class="form-control form-input form-input-bordered"  :required="field.attribute === 'source'">
                </td>
              </tr>
            </tbody>
          </table>
        </template>
  
        <div class="flex justify-center space-x-2">
          <LinkButton @click="goBack">&leftarrow; Upload a different file</LinkButton>
          <DefaultButton :disabled="disabledSave" @click="saveConfig">{{ saving ? 'Importing...' : 'Save &amp; Preview &rightarrow;' }}</DefaultButton>
        </div>
      </card>
    </div>
  </template>
  
  <script>
  import draggable from 'vuedraggable'
  
  export default {
    components: {
      draggable,
    },
  
    data() {
      return {
        resource: 'clients',
        mappings: this.config?.mappings || {},
        values: this.config?.values || {},
        modifiers: this.config?.modifiers || {},
        saving: false,
      };
    },
  
    props: [
      'headings',
      'resources',
      'fields',
      'file',
      'file_name',
      'rows',
      'total_rows',
      'config',
      'mods',
    ],
  
    watch: {
      resource: {
        immediate: true,
        handler(newValue) {
          if (newValue === "") {
            return;
          }
  
          const fields = this.fields[newValue];
  
          // Restore original settings
          if (newValue === this.config?.resource) {
            this.mappings = { ...this.config?.mappings };
            this.values = { ...this.config?.values };
            this.modifiers = { ...this.config?.modifiers };
  
            return;
          }
  
          // Reset all of the mappings and any custom values
          this.mappings = {};
          this.values = {};
  
          // For each field of the resource, try to find a matching heading and pre-assign
          for (let { name, attribute } of fields) {
            let heading = this.getMatchingValue(attribute);
  
            if (heading) {
              this.mappings[attribute] = heading;
              this.values[attribute] = "";
            }
          }
        },
        deep: true,
      },
    },
  
    methods: {
      updateMapping(attribute, value) {
        this.mappings[attribute] = value;
      },
  
      saveConfig() {
        if (!this.hasValidConfiguration()) {
          return;
        }
  
        this.saving = true;
  
        let data = {
          resource: this.resource,
          mappings: this.mappings,
          values: this.values,
          modifiers: this.modifiers,
          file: this.file,
        };
  
        Nova.request()
          .post(this.url('configure'), data)
          .then((response) => {
            if (response.status === 200) {
              Nova.success('Configuration saved');
              Nova.visit('/csv-import/preview/' + this.file);
            }
          })
          .catch((e) => {
            this.saving = false;
            Nova.error('There was a problem saving your configuration');
          });
  
        this.saving = false;
      },
  
      goBack() {
        Nova.visit('/csv-import/')
      },
  
      hasValidConfiguration() {
        const mappedColumns = Object.values(this.mappings);
        return this.resource !== '' && mappedColumns.length > 0;
      },
  
      url(path) {
        return '/nova-vendor/custom-import/' + path;
      },
  
      addModifier(attribute) {
        if (Array.isArray(this.modifiers[attribute])) {
          this.modifiers[attribute].push({ name: '', settings: {} });
        } else {
          this.modifiers[attribute] = [{ name: '', settings: {} }];
        }
      },
  
      getSelectedValue(field) {
        const selectedValue = this.mappings[field.attribute];
  
        if (selectedValue && this.headings.includes(selectedValue)) {
          return selectedValue;
        }
  
        return "";
      },
  
      getMatchingValue(attribute) {
        for (const heading of this.headings) {
          if (
            heading.toLowerCase().includes(attribute.toLowerCase()) ||
            attribute.toLowerCase().includes(heading.toLowerCase()) ||
            heading.toLowerCase() === attribute.toLowerCase()
          ) {
            return heading;
          }
        }
        return "";
      },
    },
  
    computed: {
      disabledSave() {
        return !this.hasValidConfiguration() || this.saving;
      },
  
      original_file_name() {
        if (this.config.original_filename?.includes('.')) {
          return this.config.original_filename.split('.').slice(0, -1).join('.');
        }
  
        return this.config.original_filename || '';
      },
    },
  };
  </script>
  