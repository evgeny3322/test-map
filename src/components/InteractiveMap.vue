<template>
  <div class="map-container">
    <div id="map" ref="mapContainer"></div>
    <div class="map-controls">
      <button @click="resetHighlights" class="control-button">Сбросить выделение</button>
    </div>
  </div>
</template>

<script>
import { ref, onMounted, watch } from 'vue';
import mapboxgl from 'mapbox-gl';
import 'mapbox-gl/dist/mapbox-gl.css';

export default {
  name: 'InteractiveMap',
  props: {
    standData: {
      type: Array,
      required: true
    }
  },
  emits: ['openStandDetails'],
  setup(props, { emit }) {
    const mapContainer = ref(null);
    const map = ref(null);
    const markers = ref([]);
    const highlightedStandId = ref(null);

    // Инициализация карты при монтировании компонента
    onMounted(() => {
      // Необходимо заменить на свой access token от Mapbox
      mapboxgl.accessToken = 'pk.eyJ1IjoiY3VybXVkZ2VvbnBoZCIsImEiOiJjbHAxZXU2dmwwajV6MmxwZzRpdXhobjB2In0.uIuuAtk1EtV7HSbjpUfeOw';

      map.value = new mapboxgl.Map({
        container: mapContainer.value,
        style: 'mapbox://styles/mapbox/light-v10',
        center: [37.622504, 55.753215], // Центр Москвы (Красная площадь)
        zoom: 10
      });

      // Добавление элементов управления к карте
      map.value.addControl(new mapboxgl.NavigationControl());

      // Ожидаем загрузку карты, прежде чем добавлять маркеры
      map.value.on('load', () => {
        addStandsToMap();
      });
    });

    // Следим за изменениями данных о стендах
    watch(() => props.standData, () => {
      if (map.value && map.value.loaded()) {
        clearMarkers();
        addStandsToMap();
      }
    });

    // Добавление стендов на карту
    const addStandsToMap = () => {
      props.standData.forEach(stand => {
        // Создаем элемент для маркера
        const el = document.createElement('div');
        el.className = 'map-marker';
        el.style.backgroundColor = stand.highlightColor || '#FF0000';
        el.textContent = stand.id;

        // Создаем popup для краткой информации при наведении
        const popup = new mapboxgl.Popup({
          offset: 25,
          closeButton: false,
          closeOnClick: false
        }).setHTML(`<strong>${stand.name}</strong>`);

        // Создаем и добавляем маркер на карту
        const marker = new mapboxgl.Marker(el)
            .setLngLat([stand.location.lng, stand.location.lat])
            .setPopup(popup)
            .addTo(map.value);

        // Добавляем обработчики событий
        el.addEventListener('mouseenter', () => {
          popup.addTo(map.value);
        });

        el.addEventListener('mouseleave', () => {
          popup.remove();
        });

        el.addEventListener('click', () => {
          highlightStand(stand.id);
          emit('openStandDetails', stand);
        });

        // Сохраняем маркер в массиве для возможности очистки
        markers.value.push({ marker, stand });
      });
    };

    // Очистка всех маркеров с карты
    const clearMarkers = () => {
      markers.value.forEach(({ marker }) => {
        marker.remove();
      });
      markers.value = [];
    };

    // Выделение выбранного стенда
    const highlightStand = (standId) => {
      // Сбрасываем предыдущее выделение
      if (highlightedStandId.value) {
        const prevMarker = markers.value.find(
            ({ stand }) => stand.id === highlightedStandId.value
        );
        if (prevMarker) {
          const el = prevMarker.marker.getElement();
          el.style.width = '30px';
          el.style.height = '30px';
          el.style.zIndex = '1';
          el.style.boxShadow = 'none';
        }
      }

      // Устанавливаем новое выделение
      highlightedStandId.value = standId;
      const currentMarker = markers.value.find(
          ({ stand }) => stand.id === standId
      );

      if (currentMarker) {
        const el = currentMarker.marker.getElement();
        el.style.width = '40px';
        el.style.height = '40px';
        el.style.zIndex = '2';
        el.style.boxShadow = '0 0 10px rgba(0, 0, 0, 0.5)';

        // Центрируем карту на выделенном стенде
        map.value.flyTo({
          center: [
            currentMarker.stand.location.lng,
            currentMarker.stand.location.lat
          ],
          zoom: 15,
          duration: 1000
        });
      }
    };

    // Сброс выделения всех стендов
    const resetHighlights = () => {
      if (highlightedStandId.value) {
        const prevMarker = markers.value.find(
            ({ stand }) => stand.id === highlightedStandId.value
        );
        if (prevMarker) {
          const el = prevMarker.marker.getElement();
          el.style.width = '30px';
          el.style.height = '30px';
          el.style.zIndex = '1';
          el.style.boxShadow = 'none';
        }
        highlightedStandId.value = null;
      }

      // Отдаляем карту для обзора всех стендов
      map.value.flyTo({
        center: [37.622504, 55.753215], // Центр карты (Красная площадь)
        zoom: 10,
        duration: 1000
      });
    };

    return {
      mapContainer,
      highlightStand,
      resetHighlights
    };
  }
};
</script>

<style>
.map-container {
  position: relative;
  width: 100%;
  height: 600px;
  border-radius: 8px;
  overflow: hidden;
  margin-bottom: 20px;
}

#map {
  width: 100%;
  height: 100%;
}

.map-marker {
  width: 30px;
  height: 30px;
  background-color: #FF0000;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
  font-weight: bold;
  cursor: pointer;
  transition: all 0.3s ease;
}

.map-marker:hover {
  transform: scale(1.1);
}

.map-controls {
  position: absolute;
  top: 10px;
  right: 10px;
  z-index: 1;
}

.control-button {
  background-color: white;
  border: 1px solid #ccc;
  border-radius: 4px;
  padding: 8px 12px;
  cursor: pointer;
  font-size: 14px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.control-button:hover {
  background-color: #f8f8f8;
}
</style>