<template>
  <div class="container">
    <div class="info-panel">
      <h3><b class="content-title">Informações da Fazenda</b></h3>
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

      <div v-if="savedAreas.length > 0" class="saved-areas">
        <div v-for="(area, index) in savedAreas" :key="index" class="card">
          <img :src="area.image" alt="Imagem da Área" class="image-card" />
          <div class="object-content">
            <p><b class="content-title">Nome:</b> {{ area.name }}</p>
            <p><b class="content-title">Área:</b> {{ area.area }} ha</p>
            <p><b class="content-title">Perímetro:</b> {{ area.perimeter }}</p>
            <p><b class="content-title">Quantidade de Pontos:</b> {{ area.points.length }}</p>
          </div>
          <button @click="editArea(index)" class="edit-btn">
            <i class="fa fa-ellipsis-v"></i>
          </button>
        </div>
      </div>

      <div class="tools-btns">
        <h4><b class="content-title">Ferramentas de Demarcação do Contorno</b></h4>
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

    <div id="map" class="map-container">
      <div class="search-bar">
        <input id="search-input" type="text" placeholder="Pesquise um endereço, cidade ou país" />
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
      autocomplete: null,
    }
  },
  methods: {
    async initMap() {
      const loader = new Loader({
        apiKey: 'AIzaSyAkCupFAHdcnfE3tBn7OyznNJ66xWSiDnM',
        libraries: ['drawing', 'geometry', 'places'],
      })

      const google = await loader.load()
      this.google = google

      this.map = new google.maps.Map(document.getElementById('map'), {
        center: { lat: -22.2176, lng: -49.6459 },
        zoom: 12,
      })

      this.drawingManager = new google.maps.drawing.DrawingManager({
        drawingMode: null,
        drawingControl: false,
        polygonOptions: { fillColor: '#FF0000', fillOpacity: 0.4, editable: true },
        rectangleOptions: { fillColor: '#0000FF', fillOpacity: 0.4, editable: true },
        circleOptions: { fillColor: '#00FF00', fillOpacity: 0.4, editable: true },
      })
      this.drawingManager.setMap(this.map)

      google.maps.event.addListener(this.drawingManager, 'overlaycomplete', (event) => {
        this.handleOverlayComplete(event)
      })

      this.initSearchBar()
    },

    handleOverlayComplete(event) {
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

      this.currentAreaSize = (area / 10000).toFixed(2)
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

      const ne = bounds.getNorthEast()
      const sw = bounds.getSouthWest()

      const points = [ne, { lat: ne.lat(), lng: sw.lng() }, sw, { lat: sw.lat(), lng: ne.lng() }]

      const perimeter = google.maps.geometry.spherical.computeLength(points)
      const area = google.maps.geometry.spherical.computeArea(points)

      this.currentAreaSize = (area / 10000).toFixed(2)
      this.currentAreaPerimeter = `${perimeter.toFixed(2)} m`
      this.currentPointCount = points.length
    },

    handleCircle(circle) {
      const google = this.google
      const radius = circle.getRadius()
      const area = Math.PI * Math.pow(radius, 2)
      const perimeter = 2 * Math.PI * radius

      this.currentAreaSize = (area / 10000).toFixed(2)
      this.currentAreaPerimeter = `${perimeter.toFixed(2)} m`
      this.currentPointCount = 1
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
      this.currentAreaName = ''
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
      const type =
        shape instanceof this.google.maps.Polygon
          ? 'POLYGON'
          : shape instanceof this.google.maps.Rectangle
            ? 'RECTANGLE'
            : 'CIRCLE'

      const savedData = {
        name: this.currentAreaName,
        area: area,
        perimeter: perimeter,
        points: points,
        type: type,
        image: this.captureMapImage(),
      }

      if (type === 'RECTANGLE') {
        savedData.bounds = {
          ne: shape.getBounds().getNorthEast().toJSON(),
          sw: shape.getBounds().getSouthWest().toJSON(),
        }
      } else if (type === 'CIRCLE') {
        savedData.center = shape.getCenter().toJSON()
        savedData.radius = shape.getRadius()
      }

      this.savedAreas.push(savedData)
      this.clearAll()
    },

    editArea(index) {
      const area = this.savedAreas[index]

      this.currentAreaName = area.name
      this.currentAreaSize = area.area
      this.currentAreaPerimeter = area.perimeter
      this.currentPointCount = area.points.length

      this.currentShapes.forEach((shape) => shape.setMap(null))
      this.currentShapes = []

      if (area.type === 'POLYGON') {
        const polygon = new this.google.maps.Polygon({
          paths: area.points,
          editable: true,
          map: this.map,
          fillColor: '#FF0000',
          fillOpacity: 0.4,
        })
        this.currentShapes.push(polygon)
        this.handlePolygon(polygon)
      } else if (area.type === 'RECTANGLE') {
        const bounds = new this.google.maps.LatLngBounds(area.bounds.sw, area.bounds.ne)
        const rectangle = new this.google.maps.Rectangle({
          bounds,
          editable: true,
          map: this.map,
          fillColor: '#0000FF',
          fillOpacity: 0.4,
        })
        this.currentShapes.push(rectangle)
        this.handleRectangle(rectangle)
      } else if (area.type === 'CIRCLE') {
        const circle = new this.google.maps.Circle({
          center: area.center,
          radius: area.radius,
          editable: true,
          map: this.map,
          fillColor: '#00FF00',
          fillOpacity: 0.4,
        })
        this.currentShapes.push(circle)
        this.handleCircle(circle)
      }
    },

    captureMapImage() {
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
          if (index === 0) return
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
            image: this.captureMapImage(),
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
.container {
  display: flex;
  flex-wrap: wrap;
  gap: 20px;
  padding: 1.75rem 2.5rem;
  border: 1px solid #e9e9e9;
  border-radius: 5px;
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
  z-index: 1000 !important;
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
  min-height: 600px;
  border: 1px solid #ddd;
  border-radius: 8px;
  overflow: hidden;
  z-index: 1 !important;
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

.saved-areas {
  width: 100%;
  display: flex;
  flex-direction: column;
  flex-wrap: wrap;
  gap: 10px;
}

.card {
  display: flex;
  justify-content: space-between;
  background: #fff;
  padding: 20px;
  border: 1px solid #ddd;
  border-radius: 8px;
  flex: 1;
  width: 100%;
}

.card img {
  width: 110px;
  height: 160px;
  border-radius: 8px;
}

.object-content {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  padding: 0rem 0.5rem;
}

.edit-btn {
  background-color: transparent;
  border: none;
  display: flex;
}

.content-title {
  font-weight: bolder;
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

.import-button input[type='file'] {
  display: none;
}
</style>
