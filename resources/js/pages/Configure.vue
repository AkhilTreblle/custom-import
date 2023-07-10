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
                                    <i v-if="! col"></i>
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
                        <tr v-for="field in fields[resource]" class="border-b">
                            <td class="pr-2">
                                <span class="font-bold">{{ field.name }}</span><br>
                                <small class="text-grey-300">{{ field.attribute }}</small>
                            </td>
                            <td class="space-y-2">
                                <SelectControl @change="(value) => mappings[field.attribute] = value" :selected="mappings[field.attribute]">
                                    <option value="" v-if="field.rules.includes('required')" disabled>- This field is required -</option>
                                    <option value="" v-else>- Leave field empty -</option>

                                    <optgroup label="File columns">
                                        <option v-for="heading in headings" :value="heading">{{ heading }}</option>
                                    </optgroup>

                                    <optgroup label="Custom - Text">
                                        <option value="custom">Text</option>
                                    </optgroup>
                                </SelectControl>

                                <input v-model="values[field.attribute]" v-if="mappings[field.attribute] === 'custom'"
                                    class="form-control form-input form-input-bordered">

                                <draggable
                                    v-model="modifiers[field.attribute]"
                                    handle=".handle"
                                    item-key="modifier">

                                    <template #item="{ element, index }">
                                        <div class="flex mb-2 space-x-2 items-start border-rounded bg-gray-50 p-2 handle">
                                            <span>{{ index + 1 }}</span>
                                            <div class="flex flex-col flex-1 space-y-2">
                                                <SelectControl @change="(value) => element.name = value" :selected="element.name">
                                                    <option value="">- Do not modify -</option>

                                                    <option v-for="mod in mods" :value="mod.name">{{ mod.title }}</option>
                                                </SelectControl>

                                                <label v-for="(config, name) in mods[element.name].settings"
                                                    v-if="mods[element.name]?.settings" class="flex items-center space-x-2"
                                                >
                                                    <span>{{ config.title }}</span>

                                                    <SelectControl v-if="config.type === 'select'"
                                                        @change="(value) => element.settings[name] = value"
                                                        :selected="element.settings[name]"
                                                    >
                                                        <option v-for="(option, value) of config.options" :value="value"
                                                            :selected="value === config.default"
                                                        >
                                                            {{ option }}
                                                        </option>
                                                    </SelectControl>

                                                    <input type="text" v-if="config.type === 'string'" v-model="element.settings[name]"
                                                        class="form-control form-input form-input-bordered ml-4" :placeholder="config.default">

                                                    <input type="text" v-if="config.type === 'boolean'" v-model="element.settings[name]"
                                                        class="checkbox" :checked="config.default">

                                                    <div class="help-text">{{ config.help }}</div>
                                                </label>
                                            </div>
                                            <button @click="removeModifier(field.attribute, index)">&times;</button>
                                        </div>
                                    </template>
                                </draggable>

                            </td>
                        </tr>
                    </tbody>
                </table>
            </template>

            <div class="flex justify-center space-x-2">
                <LinkButton @click="goBack">
                    &leftarrow; Upload a different file
                </LinkButton>
                <DefaultButton :disabled="disabledSave" @click="saveConfig">
                    {{ saving ? 'Importing...' : 'Save &amp; Preview &rightarrow;' }}
                </DefaultButton>
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
            resource:'clients',
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
                this.mappings = this.config?.mappings || {};
                this.values = this.config?.values || {};

                return;
            }

            // Reset all of the mappings and any custom values
            this.mappings = {};
            this.values = {};

            // For each field of the resource, try to find a matching heading and pre-assign
            for (let { name, attribute } of fields) {
                let heading = this.headings.indexOf(attribute);

                if (heading < 0) {
                    continue;
                }

                // Because they're an exact match, we don't need to get the exact heading out
                this.mappings[attribute] = attribute;
                this.values[attribute] = ""; // Initialize custom value to empty string
            }
        },
        deep: true,
    }
},

    methods: {
        removeModifier(attribute, index) {
            this.modifiers[attribute].splice(index, 1);
        },

        saveConfig() {
            if (! this.hasValidConfiguration()) {
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
            const mappedColumns = [],
                mappings = this.mappings;

            Object.keys(mappings).forEach(function (key) {
                if (mappings[key] !== "") {
                    mappedColumns.push(key);
                }
            });

            return this.resource !== '' && mappedColumns.length > 0;
        },

        url(path) {
            return '/nova-vendor/custom-import/' + path;
        },

        addModifier(attribute) {
            if (Array.isArray(this.modifiers[attribute])) {
                this.modifiers[attribute].push({name: '', settings: {}});
                return;
            }

            this.modifiers[attribute] = [{name: '', settings: {}}];
        },
    },

    computed: {
        disabledSave() {
            return ! this.hasValidConfiguration() || this.saving;
        },

        original_file_name() {
            if (this.config.original_filename?.includes('.')) {
                return this.config.original_filename.split('.').slice(0, -1).join('.');
            }

            return this.config.original_filename || '';
        }
    }
}
</script>