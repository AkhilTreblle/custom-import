<template>
  <div>
    <Head>
      <title>Import Data</title>
    </Head>
      <h1 class="text-3xl font-bold mb-6">CSV Import</h1>

    <div class="bg-white dark:bg-gray-800 rounded-lg shadow">

      <div class="w-96 mx-auto">
        <div class="rounded-lg shadow-md p-6">
          <label for="file" class="block text-lg font-medium mb-3">Select CSV File:</label>
          <input type="file" id="file" name="file" ref="file" @change="handleFile" class="mb-3">

          <button v-bind:disabled="!file" @click="upload" class="shadow relative bg-primary-500 hover:bg-primary-400 text-white dark:text-gray-900 cursor-pointer rounded text-sm font-bold focus:outline-none focus:ring ring-primary-200 dark:ring-gray-600 inline-flex items-center justify-center h-9 px-3 shadow relative bg-primary-500 hover:bg-primary-400 text-white dark:text-gray-900">
            Upload & Configure
          </button>
        </div>
      </div>
    </div>
  </div>
</template>


<script>
export default {
    data() {
        return {
            file: '',
        };
    },

    methods: {
        handleFile: function (event) {
            this.file = this.$refs.file.files[0];
        },

        upload: function (event) {
            let formData = new FormData();
            // send it to the server
            formData.append('file', this.file);

            const self = this;

            // if it's valid, move to the next screen
            return Nova.request()
                .post('/nova-vendor/custom-import/upload',
                    formData,
                    {
                        headers: {
                            'Content-Type': 'multipart/form-data'
                        }
                    }
                )
                .then((response) => {
                    Nova.success('File uploaded!')
                    Nova.visit(response.data.configure);
                })
                .catch((e) => {
                    Nova.error(e.response.data.message);
                });
        }
    }
}
</script>
