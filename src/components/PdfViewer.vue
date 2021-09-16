<template>
  <div>
    <div style="display: flex; flex-flow: row wrap">
      <div style="margin: 10px">
        <button @click="createInput()" :disabled="!pdfFile">
          create input
        </button>
      </div>
      <div style="margin: 10px">
        <button @click="logmetadata()" :disabled="!pdfFile">
          log metadada
        </button>
      </div>
      <div style="margin: 10px">
        <button @click="showFormFields()" :disabled="!pdfFile">
          show and fill form fields
        </button>
      </div>
      <div style="margin: 10px">
        <button @click="canvasToImg()" :disabled="!pdfFile">
          convert to img
        </button>
      </div>

      <div style="margin: 10px">
        <b style="margin: 10px">steps input</b>
        <input type="number" v-model="steps" placeholder="edíteme" />
      </div>
      <div style="margin: 10px">
        <input
          type="file"
          id="pdf"
          name="pdf"
          v-on:change="pdfSeter"
          accept="application/pdf,application/vnd.ms-excel"
        />
      </div>
    </div>
    <button class="download" @click="addInputsToPdf()" :disabled="!pdfFile">
      ↓
    </button>
    <button
      style="margin: 10px;"
      @click="() => (inputList = [])"
      :disabled="!pdfFile"
    >
      empty input list
    </button>
    <button style="margin: 10px;" @click="removeFields()" :disabled="!pdfFile">
      remove default field list
    </button>
    <div style="display: flex">
      <div>
        <canvas id="my_canvas"></canvas>
      </div>
      <div>
        <div>
          <h3>
            default form field list
            <button
              style="margin: 10px;"
              @click="showDefaultFormFields = !showDefaultFormFields"
              :disabled="!pdfFile"
            >
              show/hide
            </button>
          </h3>
          <div v-if="showDefaultFormFields">
            <p
              v-for="(input, i) in defaultFormFields"
              :key="i"
              style="margin: 10px"
            >
              {{ input }}
              <button
                style="margin: 10px;"
                @click="removeFields(input.name)"
                :disabled="!pdfFile"
              >
                remove field
              </button>
            </p>
          </div>
        </div>
        <div>
          <h3>input list</h3>
          <div style="display: flex; flex-flow: row wrap">
            <div v-for="(input, i) in inputList" :key="i" style="margin: 10px">
              <div>
                name:
                <input
                  style="margin: 10px; width: 94px"
                  id="name"
                  name="name"
                  v-model="input.inputName"
                />
              </div>
              <div style="display: flex; flex-direction: row">
                <input
                  type="number"
                  id="height"
                  name="height"
                  min="1"
                  max="1000"
                  :step="steps"
                  v-model="input.height"
                />
                <input
                  type="number"
                  id="top"
                  name="top"
                  min="1"
                  max="1000"
                  :step="steps"
                  v-model="input.top"
                />
              </div>
              <div>
                <input
                  type="number"
                  id="width"
                  name="width"
                  min="1"
                  max="1000"
                  step="10"
                  v-model="input.width"
                />
                <input
                  type="number"
                  id="left"
                  name="left"
                  min="1"
                  max="1000"
                  :step="steps"
                  v-model="input.left"
                />
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import pdfjsLib from "pdfjs-dist/build/pdf";
import "pdfjs-dist/web/pdf_viewer.css";
import { PDFDocument, StandardFonts } from "pdf-lib";

pdfjsLib.GlobalWorkerOptions.workerSrc =
  "https://cdn.jsdelivr.net/npm/pdfjs-dist@2.0.943/build/pdf.worker.min.js";

export default {
  name: "PdfViewer",
  data() {
    return {
      url: "./pdf-sample.pdf",
      canvas: undefined,
      ctx: undefined,
      pdfFile: undefined,
      pageOfSelectedPdf: undefined,
      inputList: [],
      defaultFormFields: [],
      isRendering: false,
      steps: 10,
      pdfDoc: undefined,
      showDefaultFormFields: false,
    };
  },
  mounted() {
    // this.getPdf(this.url);
  },
  watch: {
    inputList: {
      handler() {
        this.updateCanvas();
      },
      deep: true,
    },
  },
  methods: {
    async removeFields(field) {
      const form = this.pdfDoc.getForm();
      if (field) {
        const fieldFound = form.getFields().find((x) => x.getName() === field);
        this.defaultFormFields.splice(
          this.defaultFormFields.findIndex((el) => el.name === field),
          1
        );

        try {
          form.removeField(fieldFound);
        } catch (err) {
          console.log(err);
        }
      } else {
        form.getFields().map((field) => {
          if (field.constructor.name === "PDFTextField") {
            form.removeField(field);
          }
        });
      }
      const pdfBytes = await this.pdfDoc.save({updateFieldAppearances: false});
      var blob = new Blob([pdfBytes], { type: "application/pdf" });
      var link = document.createElement("a");
      link.href = window.URL.createObjectURL(blob);
      var fileName = "myname";
      link.download = fileName;
      link.click();
    },
    async addInputsToPdf() {
      console.log("pdf will be generated by");
      console.log(JSON.stringify(this.inputList));

      const timesRomanFont = await this.pdfDoc.embedFont(
        StandardFonts.TimesRoman
      );

      const pages = this.pdfDoc.getPages();
      const firstPage = pages[0];
      const { height } = firstPage.getSize();

      const form = this.pdfDoc.getForm();
      this.inputList.map((input) => {
        const field = form.createTextField(input.inputName);
        field.addToPage(firstPage, {
          x: parseInt(input.left),
          y: height - input.height - parseInt(input.top),
          width: parseInt(input.width),
          height: parseInt(input.height),
          borderWidth: 0,
          font: timesRomanFont,
        });
        input.size && field.setFontSize(input.size);
        input.multiline && field.enableMultiline();
      });

      const pdfBytes = await this.pdfDoc.save();
      var blob = new Blob([pdfBytes], { type: "application/pdf" });
      var link = document.createElement("a");
      link.href = window.URL.createObjectURL(blob);
      var fileName = "myname";
      link.download = fileName;
      link.click();
    },
    async pdfSeter(file) {
      this.pdfFile = file.target.files[0];
      this.pdfFile.arrayBuffer().then((buffer) => {
        this.getPdf(buffer);
        PDFDocument.load(buffer).then((doc) => (this.pdfDoc = doc));
      });
    },
    canvasToImg() {
      const imgURL = this.canvas.toDataURL("image/png");
      var dlLink = document.createElement("a");
      var MIME_TYPE = "image/png";

      dlLink.download = "imaagennn";
      dlLink.href = imgURL;
      dlLink.dataset.downloadurl = [
        MIME_TYPE,
        dlLink.download,
        dlLink.href,
      ].join(":");

      document.body.appendChild(dlLink);
      dlLink.click();
      document.body.removeChild(dlLink);
    },
    async getPdf(buffer) {
      let loadingTask = pdfjsLib.getDocument(buffer);
      let pdf = await loadingTask.promise;
      pdf.getPage(1).then((page) => {
        this.canvas = document.getElementById("my_canvas");
        this.ctx = this.canvas.getContext("2d");
        var viewport = page.getViewport(1);
        this.canvas.width = viewport.width;
        this.canvas.height = viewport.height;

        this.pageOfSelectedPdf = page;
        this.renderPdf();
      });
    },
    createInput() {
      this.inputList = [
        ...this.inputList,
        {
          width: 50,
          height: 50,
          top: 50,
          left: 50,
          size: 5,
          multiline: true,
          alignement: "Center",
          inputName: "defaultName" + this.inputList.length,
        },
      ];
      this.updateCanvas();
    },
    logmetadata() {

    },
    async showFormFields() {
      const formPdfBytes = await this.pdfFile.arrayBuffer();
      const pdfDoc = await PDFDocument.load(formPdfBytes);

      const form = pdfDoc.getForm();
      const fields = form.getFields();
      fields.forEach((field) => {
        const type = field.constructor.name;
        const name = field.getName();
        this.defaultFormFields = [...this.defaultFormFields, { type, name }];
        //fill form
        if (type === "PDFTextField") {
          const fieldToFill = form.getTextField(name);
          fieldToFill.removeMaxLength();
          fieldToFill.setText(name);
        }
      });
      const pdfBytes = await pdfDoc.save();
      var blob = new Blob([pdfBytes], { type: "application/pdf" });
      var link = document.createElement("a");
      link.href = window.URL.createObjectURL(blob);
      var fileName = "myname";
      link.download = fileName;
      link.click();
    },
    renderPdf() {
      const viewport = this.pageOfSelectedPdf.getViewport(1);
      return this.pageOfSelectedPdf.render({
        canvasContext: this.ctx,
        viewport: viewport,
      });
    },
    updateCanvas() {
      if (!this.isRendering) {
        //borra todo
        this.isRendering = true;
        this.renderPdf().then(() => {
          //cuando vuelve a renderizar el pdf añadimos los inputs a canvas
          this.inputList.map((input) => {
            this.ctx.strokeRect(
              input.left,
              input.top,
              input.width,
              input.height
            );
          });
          this.isRendering = false;
        });
      }
    },
  },
};
</script>

<style>
.download {
  text-decoration: none;
  cursor: pointer;
  border: none;
  background-color: teal;
  position: absolute;
  right: 0;
  top: 0;
  margin: 10px 20px;
  padding: 3px 11px;
  padding-bottom: 8px;
  border-radius: 5px;
  color: white;
}
</style>
