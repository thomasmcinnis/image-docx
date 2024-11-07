<template>
  <h1>Pokemon Document Generator</h1>
  <button @click="renderPokemonImage">Show Pokemon Image</button>
  <button @click="downloadDocWithUrl">Download Doc (URL method)</button>
  <button @click="downloadDocWithBase64String">Download Doc (Base64 method)</button>

  <div v-if="imageElement">
    <img :src="imageElement.src" />
  </div>
</template>

<script>
import { Document, ImageRun, Header, Packer, Paragraph } from "docx";

export default {
  name: 'PokemonDoc',

  data() {
    return {
      imageElement: null
    }
  },

  methods: {
    createImageHtmlElement(base64String) {
      const img = new Image()
      img.src = base64String
      img.onload = () => {
        this.imageElement = img
      }
    },

    async getPokemonData() {
      try {
        const res = await fetch('https://pokeapi.co/api/v2/pokemon/ditto/')
        if (!res.ok) {
          throw new Error(`HTTP error! status: ${res.status}`)
        }
        const data = await res.json()
        console.log(data)
        return data
      } catch (error) {
        console.error('Error:', error)
      }
    },

    async fetchImageAsBase64(url) {
      try {
        const response = await fetch(url);
        const blob = await response.blob();
        return new Promise((resolve, reject) => {
          const reader = new FileReader();
          reader.onloadend = () => resolve(reader.result);
          reader.onerror = reject;
          reader.readAsDataURL(blob);
        });
      } catch (error) {
        console.error('Error:', error);
        throw error;
      }
    },

    async renderPokemonImage() {
      try {
        const data = await this.getPokemonData()
        const url = data.sprites.front_default
        const encodedData = await this.fetchImageAsBase64(url)
        console.log(encodedData)
        this.createImageHtmlElement(encodedData)
      } catch (err) {
        console.log(err)
      }
    },

    handleDownload(blob, filename) {
      const url = URL.createObjectURL(blob)
      const link = document.createElement('a')
      link.href = url
      link.download = filename
      document.body.appendChild(link)
      link.click()

      // Cleanup
      document.body.removeChild(link)
      URL.revokeObjectURL(url)
    },

    async downloadDocWithUrl() {
      const data = await this.getPokemonData()
      const { name, weight, height } = data
      const characterHeader = `Name: ${name} | Weight: ${weight} | Height: ${height}`
      const url = data.sprites.front_default
      const blob = await fetch(url).then(res => res.blob())
      const doc = new Document({
        sections: [
          {
            headers: {
              default: new Header({
                children: [new Paragraph(characterHeader)]
              })
            },
            children: [
              new Paragraph("Front Image"),
              new Paragraph({
                children: [
                  new ImageRun({
                    data: blob,
                    transformation: {
                      width: 100,
                      height: 100
                    }
                  })
                ]
              })
            ]
          }

        ]
      })

      Packer.toBlob(doc).then(blob => {
        this.handleDownload(blob, 'my-pokemon.docx')
      })
    },

    async downloadDocWithBase64String() {
      const data = await this.getPokemonData()
      const { name, weight, height } = data
      const characterHeader = `Name: ${name} | Weight: ${weight} | Height: ${height}`
      const url = data.sprites.front_default
      const imageBase64Data = await this.fetchImageAsBase64(url)
      const base64String = imageBase64Data.replace(/^data:image\/\w+;base64,/, '')
      const doc = new Document({
        sections: [
          {
            headers: {
              default: new Header({
                children: [new Paragraph(characterHeader)]
              })
            },
            children: [
              new Paragraph("FrontImage"),
              new Paragraph({
                children: [
                  new ImageRun({
                    data: Uint8Array.from(atob(base64String), c => c.charCodeAt(0)),
                    transformation: {
                      width: 100,
                      height: 100
                    }
                  })
                ]
              })
            ]
          }
        ]
      })

      Packer.toBlob(doc).then(blob => {
        this.handleDownload(blob, 'my-pokemon.docx')
      })
    }
  }
}
</script>
