<template>
  <div class="container">
    <div class="info-panel">
      <h3>Informações da Fazenda</h3>
      <div class="info-coloumn">
        <div class="info-row">
          <div class="info-box">
            <label for="area-name">Nome</label>
            <input v-model="currentAreaName" id="area-name" placeholder="Digite o nome da área" />
          </div>
          <div class="info-box">
            <label for="area-size">Área (ha)</label>
            <input v-model="currentAreaSize" id="area-size" disabled />
          </div>
        </div>
        <div class="info-row">
          <div class="info-box">
            <label for="area-perimeter">Perímetro</label>
            <input v-model="currentAreaPerimeter" id="area-perimeter" disabled />
          </div>
          <div class="info-box">
            <label for="area-points">Quantidade de Pontos</label>
            <input v-model="currentPointCount" id="area-points" disabled />
          </div>
        </div>
      </div>

      <div class="tools-btns">
        <h4>Ferramentas de Demarcação do Contorno</h4>
        <div class="tools">
          <div class="tools-row">
            <div class="btn-coloumn">
              <button @click="setDrawingMode('CIRCLE')">
                <i class="fa fa-circle-thin"></i>
              </button>
              <span class="tools-name">Circulo</span>
            </div>
            <div class="btn-coloumn">
              <button @click="setDrawingMode('RECTANGLE')">
                <i class="fa fa-square-o"> </i>
              </button>
              <span class="tools-name">Retângulo</span>
            </div>
            <div class="btn-coloumn">
              <button @click="setDrawingMode('POLYGON')">
                <i class="fa fa-pencil-square-o"></i>
              </button>
              <span class="tools-name">Livre</span>
            </div>
            <div class="btn-coloumn">
              <label class="import-button">
                <i class="fa fa-upload"></i>
                <input type="file" @change="importCSV" accept=".csv" />
              </label>
              <span class="tools-name">Importar</span>
            </div>
          </div>

          <div class="tools-row">
            <div class="btn-coloumn">
              <button @click="undoLastPoint">
                <i class="fa fa-undo"></i>
              </button>
              <span class="tools-name">Desfazer o ponto</span>
            </div>
            <div class="btn-coloumn">
              <button @click="clearAll">
                <i class="fa fa-eraser"></i>
              </button>
              <span class="tools-name">Limpar o desenho</span>
            </div>
            <div class="btn-coloumn">
              <button @click="saveArea" :disabled="!currentAreaName">
                <i class="fa fa-floppy-o"></i>
              </button>
              <span class="tools-name">Salvar</span>
            </div>
            <div class="btn-coloumn">
              <button @click="exportToCSV" :disabled="savedAreas.length === 0">
                <i class="fa fa-download"></i>
              </button>
              <span class="tools-name"> Exportar o CSV</span>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Map Container -->
    <div id="map" class="map-container">
      <div class="search-bar">
        <input id="search-input" type="text" placeholder="Pesquise um endereço, cidade ou país" />
      </div>
    </div>

    <!-- Cards das Áreas Salvas -->
    <div v-if="savedAreas.length > 0" class="saved-areas">
      <div v-for="(area, index) in savedAreas" :key="index" class="card">
        <h4>{{ area.name }}</h4>
        <p>Área: {{ area.area }} ha</p>
        <p>Perímetro: {{ area.perimeter }}</p>
        <p>Quantidade de Pontos: {{ area.points.length }}</p>
        <img :src="area.image" alt="Imagem da Área" class="image-card" />
      </div>
    </div>
  </div>
</template>
<script>
import { Loader } from '@googlemaps/js-api-loader'

export default {
  data() {
    return {
      map: null,
      drawingManager: null,
      currentShapes: [],
      currentAreaName: '',
      currentAreaSize: '',
      currentAreaPerimeter: '',
      currentPointCount: 0,
      savedAreas: [],
      autocomplete: null, // Autocomplete handler
    }
  },
  methods: {
    async initMap() {
      const loader = new Loader({
        apiKey: 'AIzaSyAkCupFAHdcnfE3tBn7OyznNJ66xWSiDnM', // Substitua pela sua chave válida
        libraries: ['drawing', 'geometry', 'places'],
      })

      const google = await loader.load()
      this.google = google

      // Inicializando o mapa
      this.map = new google.maps.Map(document.getElementById('map'), {
        center: { lat: -22.2176, lng: -49.6459 },
        zoom: 12,
      })

      // Configurando o gerenciador de desenhos
      this.drawingManager = new google.maps.drawing.DrawingManager({
        drawingMode: null,
        drawingControl: false,
        polygonOptions: { fillColor: '#FF0000', fillOpacity: 0.4, editable: true },
        rectangleOptions: { fillColor: '#0000FF', fillOpacity: 0.4, editable: true },
        circleOptions: { fillColor: '#00FF00', fillOpacity: 0.4, editable: true },
      })
      this.drawingManager.setMap(this.map)

      // Eventos para formas desenhadas
      google.maps.event.addListener(this.drawingManager, 'overlaycomplete', (event) => {
        this.handleOverlayComplete(event)
      })

      // Configurando o campo de busca
      this.initSearchBar()
    },

    handleOverlayComplete(event) {
      // Limpar formas anteriores, mantendo o desenho atual
      this.currentShapes.forEach((shape) => shape.setMap(null))
      this.currentShapes = [event.overlay]

      if (event.type === 'polygon') {
        this.handlePolygon(event.overlay)
      } else if (event.type === 'rectangle') {
        this.handleRectangle(event.overlay)
      } else if (event.type === 'circle') {
        this.handleCircle(event.overlay)
      }
    },

    handlePolygon(polygon) {
      const google = this.google
      this.updatePolygonInfo(polygon)
      const path = polygon.getPath()

      path.addListener('set_at', () => this.updatePolygonInfo(polygon))
      path.addListener('insert_at', () => this.updatePolygonInfo(polygon))
      path.addListener('remove_at', () => this.updatePolygonInfo(polygon))
    },

    updatePolygonInfo(polygon) {
      const google = this.google
      const area = google.maps.geometry.spherical.computeArea(polygon.getPath())
      const perimeter = google.maps.geometry.spherical.computeLength(polygon.getPath())
      const pointCount = polygon.getPath().getLength()

      this.currentAreaSize = (area / 10000).toFixed(2) // Convertendo m² para hectares
      this.currentAreaPerimeter = `${perimeter.toFixed(2)} m`
      this.currentPointCount = pointCount
    },

    handleRectangle(rectangle) {
      const google = this.google
      this.updateRectangleInfo(rectangle)

      google.maps.event.addListener(rectangle, 'bounds_changed', () => {
        this.updateRectangleInfo(rectangle)
      })
    },

    updateRectangleInfo(rectangle) {
      const google = this.google
      const bounds = rectangle.getBounds()

      const ne = bounds.getNorthEast() // Nordeste
      const sw = bounds.getSouthWest() // Sudoeste

      // Calcula os 4 pontos do retângulo
      const points = [ne, { lat: ne.lat(), lng: sw.lng() }, sw, { lat: sw.lat(), lng: ne.lng() }]

      const perimeter = google.maps.geometry.spherical.computeLength(points)
      const area = google.maps.geometry.spherical.computeArea(points)

      this.currentAreaSize = (area / 10000).toFixed(2) // m² para hectares
      this.currentAreaPerimeter = `${perimeter.toFixed(2)} m`
      this.currentPointCount = points.length
    },

    handleCircle(circle) {
      const google = this.google
      const radius = circle.getRadius()
      const area = Math.PI * Math.pow(radius, 2)
      const perimeter = 2 * Math.PI * radius

      this.currentAreaSize = (area / 10000).toFixed(2) // m² para hectares
      this.currentAreaPerimeter = `${perimeter.toFixed(2)} m`
      this.currentPointCount = 1 // Apenas o centro do círculo
    },

    setDrawingMode(mode) {
      const google = this.google
      const modes = {
        CIRCLE: google.maps.drawing.OverlayType.CIRCLE,
        RECTANGLE: google.maps.drawing.OverlayType.RECTANGLE,
        POLYGON: google.maps.drawing.OverlayType.POLYGON,
      }
      this.drawingManager.setDrawingMode(modes[mode])
    },

    clearAll() {
      this.currentShapes.forEach((shape) => shape.setMap(null))
      this.currentShapes = []
      this.currentAreaSize = ''
      this.currentAreaPerimeter = ''
      this.currentPointCount = 0
    },

    saveArea() {
      const shape = this.currentShapes[0]
      if (!shape) return

      const points = shape.getPath
        ? shape
            .getPath()
            .getArray()
            .map((point) => ({ lat: point.lat(), lng: point.lng() }))
        : []

      const area = this.currentAreaSize
      const perimeter = this.currentAreaPerimeter

      // Adiciona a nova área à lista de áreas salvas
      this.savedAreas.push({
        name: this.currentAreaName,
        area: area,
        perimeter: perimeter,
        points: points,
        image: this.captureMapImage(), // Placeholder para imagem
      })

      // Limpar dados para novo desenho
      this.clearAll()
    },

    captureMapImage() {
      // Função de captura da imagem (apenas como placeholder)
      return 'https://via.placeholder.com/200'
    },

    exportToCSV() {
      const csvRows = ['Nome,Área (ha),Perímetro,Pontos (Coordenadas)']
      this.savedAreas.forEach((area) => {
        const points = area.points.map((point) => `${point.lat},${point.lng}`).join('; ')

        csvRows.push(`${area.name},${area.area},${area.perimeter},${points}`)
      })

      const csvString = csvRows.join('\n')
      const blob = new Blob([csvString], { type: 'text/csv' })
      const link = document.createElement('a')
      link.href = URL.createObjectURL(blob)
      link.download = 'areas_salvas.csv'
      link.click()
    },

    importCSV(event) {
      const file = event.target.files[0]
      if (!file) return

      const reader = new FileReader()
      reader.onload = (e) => {
        const csvContent = e.target.result
        const rows = csvContent.split('\n')

        rows.forEach((row, index) => {
          if (index === 0) return // Ignorar cabeçalho
          const columns = row.split(',')

          const points = columns[3]
            ? points.split('; ').map((point) => {
                const [lat, lng] = point.split(',')
                return { lat: parseFloat(lat), lng: parseFloat(lng) }
              })
            : []

          this.savedAreas.push({
            name: columns[0],
            area: columns[1],
            perimeter: columns[2],
            points: points,
            image: this.captureMapImage(), // Placeholder para a imagem
          })
        })
      }

      reader.readAsText(file)
    },
  },

  mounted() {
    this.initMap()
  },
}
</script>

<style>
/* O mesmo estilo de antes */
.container {
  display: flex;
  flex-wrap: wrap;
  gap: 20px;
}

.info-panel {
  flex: 1;
  padding: 20px;
  border: 1px solid #ddd;
  border-radius: 8px;
  background: #f9f9f9;
}

.info-panel h3,
.info-panel h4 {
  margin: 0 0 10px;
}

.info-panel h4 {
  background-color: rgb(201, 196, 196);
  border-radius: 3px;
}

.info-panel label {
  font-weight: bold;
}

.tools-btns {
  border: 1px solid rgb(201, 196, 196);
  border-radius: 3px;
}

.tools {
  padding: 1rem;
}

.tools-row {
  display: flex;
  flex-direction: row;
  justify-content: space-between;
  width: 100%;
}

.btn-coloumn {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 60px;
}

.tools-name {
  font-size: 12px;
}

.info-column {
  display: flex;
  flex-direction: column;
}

.info-row {
  display: flex;
  flex-direction: row;
  justify-content: space-between;
  gap: 20px;
  margin-bottom: 2rem;
}

.info-box {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
}

#area-name,
#area-size,
#area-perimeter,
#area-points {
  padding: 0.5rem 0.75rem;
  border: 1px solid #ddd;
  border-radius: 0.25rem;
}

.search-bar {
  position: absolute !important;
  top: 10px;
  left: 50%;
  transform: translateX(-50%);
  z-index: 1000 !important; /* Aumenta o valor do z-index para garantir que fique sobre o mapa */
  width: 300px;
}

.search-bar input {
  width: 100%;
  padding: 10px;
  font-size: 14px;
  border: 1px solid #ddd;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  background-color: white;
}

.search-input {
  position: absolute !important;
  z-index: 100 !important;
}

.map-container {
  position: relative;
  flex: 3;
  height: 600px;
  border: 1px solid #ddd;
  border-radius: 8px;
  overflow: hidden;
  z-index: 1 !important; /* Garante que o mapa tenha um z-index menor que o da search bar */
}

.tools {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  margin-top: 10px;
}

.btn-coloumn > button {
  padding: 0.75rem 1rem;
  color: black;
  width: 60px;
  /* font-size: 14px; */
  cursor: pointer;
  background-color: #e9e9e9;
  border: none;
  border-radius: 4px;
}

button:hover {
  background-color: rgb(126, 123, 123);
}

.fa {
  font-size: 20px !important;
  color: rgba(63, 63, 70, 1);
}

button:disabled {
  cursor: not-allowed;
}

/* Estilo para os cards das áreas salvas */
.saved-areas {
  width: 100%;
  display: flex;
  flex-wrap: wrap;
  gap: 20px;
}

.card {
  background: #fff;
  padding: 20px;
  border: 1px solid #ddd;
  border-radius: 8px;
  flex: 1;
  max-width: 200px;
}

.card img {
  width: 100%;
  height: auto;
  border-radius: 8px;
  margin-top: 10px;
}

.import-button {
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: #e9e9e9;
  padding: 0.75rem 1rem;
  cursor: pointer;
  border: none;
  border-radius: 4px;
  transition: background 0.3s;
}

.import-button:hover {
  background-color: rgb(126, 123, 123);
}

.import-button i {
  font-size: 16px;
}

/* Esconde o input original */
.import-button input[type='file'] {
  display: none;
}
</style>
