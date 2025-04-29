<template>
  <div class="map-container">
    <div id="map" ref="mapContainer"></div>
    <div class="map-controls">
      <button @click="resetHighlights" class="control-button">Сбросить выделение</button>
      <button @click="toggleBuildings" class="control-button" v-if="showBuildingsToggle">
        {{ show3DBuildings ? 'Скрыть 3D-здания' : 'Показать 3D-здания' }}
      </button>
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
    },
    mapMarkers: {
      type: Array,
      default: () => []
    }
  },
  emits: ['openStandDetails'],
  setup(props, { emit }) {
    const mapContainer = ref(null);
    const map = ref(null);
    const markers = ref([]);
    const highlightedAreaId = ref(null);
    const mapAreas = ref([]);
    const show3DBuildings = ref(false);
    const showBuildingsToggle = ref(false);

    // Инициализация карты при монтировании компонента
    onMounted(() => {
      // Необходимо заменить на свой access token от Mapbox
      mapboxgl.accessToken = 'pk.eyJ1IjoiY3VybXVkZ2VvbnBoZCIsImEiOiJjbHAxZXU2dmwwajV6MmxwZzRpdXhobjB2In0.uIuuAtk1EtV7HSbjpUfeOw';

      map.value = new mapboxgl.Map({
        container: mapContainer.value,
        style: 'mapbox://styles/mapbox/light-v10',
        center: [37.634255, 55.832910], // Центр ВДНХ
        zoom: 15,
        pitch: 45, // наклон карты для 3D-эффекта
        bearing: -10 // поворот карты
      });

      // Добавление элементов управления к карте
      map.value.addControl(new mapboxgl.NavigationControl());

      // Ожидаем загрузку карты, прежде чем добавлять элементы
      map.value.on('load', () => {
        // Добавляем 3D-здания
        add3DBuildings();

        // Устанавливаем начальную видимость 3D-зданий
        toggleBuildingsVisibility();

        // Добавляем области и маркеры
        addAreasToMap();
        addCustomMarkersToMap();
      });
    });

    // Следим за изменениями данных о стендах
    watch(() => props.standData, () => {
      if (map.value && map.value.loaded()) {
        clearMapElements();
        addAreasToMap();
      }
    });

    // Следим за изменениями маркеров
    watch(() => props.mapMarkers, () => {
      if (map.value && map.value.loaded()) {
        clearMarkers();
        addCustomMarkersToMap();
      }
    });

    // Добавление 3D-слоя зданий
    const add3DBuildings = () => {
      try {
        map.value.addLayer({
          'id': '3d-buildings',
          'source': 'composite',
          'source-layer': 'building',
          'filter': ['==', 'extrude', 'true'],
          'type': 'fill-extrusion',
          'minzoom': 15,
          'paint': {
            'fill-extrusion-color': '#aaa',
            'fill-extrusion-height': [
              'interpolate', ['linear'], ['zoom'],
              15, 0,
              15.05, ['get', 'height']
            ],
            'fill-extrusion-base': [
              'interpolate', ['linear'], ['zoom'],
              15, 0,
              15.05, ['get', 'min_height']
            ],
            'fill-extrusion-opacity': 0.6
          }
        });
        showBuildingsToggle.value = true;
      } catch (error) {
        console.error('Ошибка при добавлении 3D-зданий:', error);
        showBuildingsToggle.value = false;
      }
    };

    // Переключение видимости 3D-зданий
    const toggleBuildings = () => {
      show3DBuildings.value = !show3DBuildings.value;
      toggleBuildingsVisibility();
    };

    // Установка видимости 3D-зданий
    const toggleBuildingsVisibility = () => {
      if (map.value && map.value.getLayer('3d-buildings')) {
        map.value.setLayoutProperty(
            '3d-buildings',
            'visibility',
            show3DBuildings.value ? 'visible' : 'none'
        );
      }
    };

    // Добавление интерактивных областей на карту
    const addAreasToMap = () => {
      // Сортируем стенды по zIndex, чтобы сначала рисовать нижние слои
      const sortedStands = [...props.standData].sort((a, b) => {
        return (a.zIndex || 1) - (b.zIndex || 1);
      });

      sortedStands.forEach((stand, index) => {
        if (stand.shape) {
          // Обрабатываем стенды с геометрической формой
          const id = `area-${stand.id}`;
          const sourceId = `source-${stand.id}`;

          // Определяем геометрию на основе типа фигуры
          const geometry = generateGeometry(stand.shape);

          // Создаем источник данных
          if (!map.value.getSource(sourceId)) {
            map.value.addSource(sourceId, {
              'type': 'geojson',
              'data': {
                'type': 'Feature',
                'properties': {
                  'name': stand.name,
                  'standId': stand.id,
                  'description': stand.description,
                  'highlightColor': stand.highlightColor || stand.shape.style.fillColor,
                  'isClickable': stand.isClickable !== false, // По умолчанию true, если не указано обратное
                  'zIndex': stand.zIndex || 1
                },
                'geometry': geometry
              }
            });
          }

          // Добавляем слой с областью
          if (!map.value.getLayer(id)) {
            map.value.addLayer({
              'id': id,
              'type': 'fill',
              'source': sourceId,
              'layout': {},
              'paint': {
                'fill-color': stand.shape.style.fillColor || '#000000',
                'fill-opacity': stand.shape.style.fillOpacity || 0.4
              }
            });

            // Добавляем контур
            map.value.addLayer({
              'id': `${id}-outline`,
              'type': 'line',
              'source': sourceId,
              'layout': {},
              'paint': {
                'line-color': stand.shape.style.strokeColor || '#000000',
                'line-width': stand.shape.style.strokeWidth || 2
              }
            });

            // Сохраняем информацию о слое
            mapAreas.value.push({
              id,
              outlineId: `${id}-outline`,
              sourceId,
              standId: stand.id,
              isClickable: stand.isClickable !== false,
              zIndex: stand.zIndex || 1
            });

            // Добавляем обработчики событий только для кликабельных областей
            if (stand.isClickable !== false) {
              map.value.on('click', id, (e) => {
                const features = map.value.queryRenderedFeatures(e.point, { layers: [id] });
                if (features.length > 0) {
                  const standId = features[0].properties.standId;
                  const stand = props.standData.find(s => s.id === standId);
                  if (stand) {
                    highlightArea(stand.id);
                    emit('openStandDetails', stand);
                  }
                }
              });

              // Изменение курсора при наведении только для кликабельных областей
              map.value.on('mouseenter', id, () => {
                map.value.getCanvas().style.cursor = 'pointer';
              });

              map.value.on('mouseleave', id, () => {
                map.value.getCanvas().style.cursor = '';
              });
            }
          }
        } else if (stand.location) {
          // Обрабатываем стенды с точечными координатами
          // Создаем элемент для маркера
          const el = document.createElement('div');
          el.className = 'map-marker';
          el.style.backgroundColor = stand.highlightColor || '#FF0000';
          el.textContent = typeof stand.id === 'number' ? stand.id : '';

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
        }
      });
    };

    // Генерация геометрии для GeoJSON на основе типа фигуры
    const generateGeometry = (shape) => {
      switch (shape.type) {
        case 'rectangle':
          // Прямоугольник определяется двумя точками: юго-западный и северо-восточный углы
          const [sw, ne] = shape.coordinates;
          return {
            'type': 'Polygon',
            'coordinates': [[
              [sw[0], sw[1]],  // Юго-западный
              [ne[0], sw[1]],  // Юго-восточный
              [ne[0], ne[1]],  // Северо-восточный
              [sw[0], ne[1]],  // Северо-западный
              [sw[0], sw[1]]   // Замыкаем полигон
            ]]
          };

        case 'polygon':
          // Полигон определяется массивом точек
          return {
            'type': 'Polygon',
            'coordinates': [
              [...shape.coordinates, shape.coordinates[0]] // Замыкаем полигон, добавляя первую точку в конец
            ]
          };

        case 'circle':
          // Круг аппроксимируется полигоном
          const points = 64; // Количество точек для аппроксимации
          const { coordinates, radius } = shape;
          const [lng, lat] = coordinates;

          // Вычисляем смещение в градусах для заданного радиуса в метрах
          // Приблизительно 111,320 метров на градус по широте
          // Для долготы расстояние зависит от широты
          const latOffset = radius / 111320;
          const lngOffset = radius / (111320 * Math.cos(lat * Math.PI / 180));

          const vertices = [];
          for (let i = 0; i < points; i++) {
            const angle = (i / points) * 2 * Math.PI;
            const dx = Math.cos(angle) * lngOffset;
            const dy = Math.sin(angle) * latOffset;
            vertices.push([lng + dx, lat + dy]);
          }
          vertices.push(vertices[0]); // Замыкаем полигон

          return {
            'type': 'Polygon',
            'coordinates': [vertices]
          };

        case 'semicircle':
          // Полукруг аппроксимируется полигоном
          const semiPoints = 32; // Количество точек для половины круга
          const { coordinates: semiCoords, radius: semiRadius, orientation } = shape;
          const [semiLng, semiLat] = semiCoords;

          // Вычисляем смещение в градусах для заданного радиуса в метрах
          const semiLatOffset = semiRadius / 111320;
          const semiLngOffset = semiRadius / (111320 * Math.cos(semiLat * Math.PI / 180));

          // Определяем начальный и конечный углы в зависимости от ориентации
          let startAngle, endAngle;
          switch (orientation) {
            case 'north':
              startAngle = Math.PI; endAngle = 2 * Math.PI;
              break;
            case 'east':
              startAngle = Math.PI * 1.5; endAngle = Math.PI * 0.5;
              break;
            case 'south':
              startAngle = 0; endAngle = Math.PI;
              break;
            case 'west':
              startAngle = Math.PI * 0.5; endAngle = Math.PI * 1.5;
              break;
            default:
              startAngle = 0; endAngle = Math.PI; // По умолчанию открытая часть на юг
          }

          // Строим полукруг
          const semiVertices = [];
          // Добавляем центральную точку для полукруга
          semiVertices.push([semiLng, semiLat]);

          // Добавляем точки дуги
          for (let i = 0; i <= semiPoints; i++) {
            const angle = startAngle + (i / semiPoints) * (endAngle - startAngle);
            const dx = Math.cos(angle) * semiLngOffset;
            const dy = Math.sin(angle) * semiLatOffset;
            semiVertices.push([semiLng + dx, semiLat + dy]);
          }

          // Замыкаем полигон
          semiVertices.push([semiLng, semiLat]);

          return {
            'type': 'Polygon',
            'coordinates': [semiVertices]
          };

        default:
          // По умолчанию возвращаем точку
          return {
            'type': 'Point',
            'coordinates': shape.coordinates
          };
      }
    };

    // Добавление SVG-маркеров на карту
    const addCustomMarkersToMap = () => {
      props.mapMarkers.forEach(markerData => {
        if (!markerData || !markerData.coordinates) return;

        // Создаем элемент для SVG маркера
        const el = document.createElement('div');
        el.className = 'map-icon';
        el.style.width = `${markerData.size || 40}px`;
        el.style.height = `${markerData.size || 40}px`;
        if (markerData.icon) {
          el.innerHTML = markerData.icon;
        }

        // Создаем попап для краткой информации при наведении
        const popup = new mapboxgl.Popup({
          offset: 25,
          closeButton: false,
          closeOnClick: false
        }).setHTML(`<strong>${markerData.name || 'Маркер'}</strong>`);

        // Создаем и добавляем маркер на карту
        const marker = new mapboxgl.Marker(el)
            .setLngLat(markerData.coordinates)
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
          // Здесь можно добавить действия при клике на иконку
          console.log(`Нажата иконка: ${markerData.name || 'Без названия'}`);
        });

        // Сохраняем маркер в массиве для возможности очистки
        markers.value.push({ marker, markerData });
      });
    };

    // Выделение выбранного стенда
    const highlightStand = (standId) => {
      // Сбрасываем предыдущее выделение
      if (highlightedAreaId.value) {
        const prevMarker = markers.value.find(
            ({ stand }) => stand && stand.id === highlightedAreaId.value
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
      highlightedAreaId.value = standId;
      const currentMarker = markers.value.find(
          ({ stand }) => stand && stand.id === standId
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

    // Выделение выбранной области
    const highlightArea = (areaId) => {
      // Сбрасываем предыдущее выделение
      if (highlightedAreaId.value) {
        const prevArea = mapAreas.value.find(area => area.standId === highlightedAreaId.value);
        if (prevArea) {
          const stand = props.standData.find(s => s.id === highlightedAreaId.value);

          if (stand && stand.shape && stand.shape.style) {
            map.value.setPaintProperty(
                prevArea.id, 'fill-color', stand.shape.style.fillColor
            );
            map.value.setPaintProperty(
                prevArea.id, 'fill-opacity', stand.shape.style.fillOpacity || 0.4
            );
            map.value.setPaintProperty(
                prevArea.outlineId, 'line-width', stand.shape.style.strokeWidth || 2
            );
          }
        }
      }

      // Устанавливаем новое выделение
      highlightedAreaId.value = areaId;
      const currentArea = mapAreas.value.find(area => area.standId === areaId);

      if (currentArea) {
        const stand = props.standData.find(s => s.id === areaId);

        if (stand && stand.shape) {
          // Увеличиваем непрозрачность и меняем цвет контура для выделения
          map.value.setPaintProperty(
              currentArea.id, 'fill-color', stand.highlightColor || stand.shape.style.fillColor
          );
          map.value.setPaintProperty(
              currentArea.id, 'fill-opacity', 0.7
          );
          map.value.setPaintProperty(
              currentArea.outlineId, 'line-width', 4
          );

          // Фокусируем карту на выделенной области
          const bounds = getBoundingBox(stand.shape);
          map.value.fitBounds(bounds, {
            padding: 100,
            duration: 1000
          });
        }
      }
    };

    // Получение ограничивающего прямоугольника для фигуры
    const getBoundingBox = (shape) => {
      switch (shape.type) {
        case 'rectangle':
          // Прямоугольник уже определен как ограничивающий прямоугольник
          return [
            shape.coordinates[0], // Юго-западный угол
            shape.coordinates[1]  // Северо-восточный угол
          ];

        case 'polygon':
          // Находим минимальные и максимальные координаты
          const lngs = shape.coordinates.map(p => p[0]);
          const lats = shape.coordinates.map(p => p[1]);
          return [
            [Math.min(...lngs), Math.min(...lats)], // Юго-западный угол
            [Math.max(...lngs), Math.max(...lats)]  // Северо-восточный угол
          ];

        case 'circle':
          // Вычисляем ограничивающий прямоугольник для круга
          const [lng, lat] = shape.coordinates;
          const latOffset = shape.radius / 111320;
          const lngOffset = shape.radius / (111320 * Math.cos(lat * Math.PI / 180));
          return [
            [lng - lngOffset, lat - latOffset], // Юго-западный угол
            [lng + lngOffset, lat + latOffset]  // Северо-восточный угол
          ];

        case 'semicircle':
          // Вычисляем ограничивающий прямоугольник для полукруга
          const [semiLng, semiLat] = shape.coordinates;
          const semiLatOffset = shape.radius / 111320;
          const semiLngOffset = shape.radius / (111320 * Math.cos(semiLat * Math.PI / 180));

          // В зависимости от ориентации меняем ограничивающий прямоугольник
          switch (shape.orientation) {
            case 'north':
              return [
                [semiLng - semiLngOffset, semiLat],
                [semiLng + semiLngOffset, semiLat + semiLatOffset]
              ];
            case 'east':
              return [
                [semiLng, semiLat - semiLatOffset],
                [semiLng + semiLngOffset, semiLat + semiLatOffset]
              ];
            case 'south':
              return [
                [semiLng - semiLngOffset, semiLat - semiLatOffset],
                [semiLng + semiLngOffset, semiLat]
              ];
            case 'west':
              return [
                [semiLng - semiLngOffset, semiLat - semiLatOffset],
                [semiLng, semiLat + semiLatOffset]
              ];
            default:
              return [
                [semiLng - semiLngOffset, semiLat - semiLatOffset],
                [semiLng + semiLngOffset, semiLat]
              ];
          }

        default:
          // По умолчанию возвращаем небольшой квадрат вокруг точки
          const [defLng, defLat] = shape.coordinates;
          return [
            [defLng - 0.001, defLat - 0.001],
            [defLng + 0.001, defLat + 0.001]
          ];
      }
    };

    // Очистка всех маркеров с карты
    const clearMarkers = () => {
      markers.value.forEach(({ marker }) => {
        marker.remove();
      });
      markers.value = [];
    };

    // Очистка всех элементов карты
    const clearMapElements = () => {
      // Удаляем слои и источники
      mapAreas.value.forEach(area => {
        if (map.value.getLayer(area.outlineId)) {
          map.value.removeLayer(area.outlineId);
        }
        if (map.value.getLayer(area.id)) {
          map.value.removeLayer(area.id);
        }
        if (map.value.getSource(area.sourceId)) {
          map.value.removeSource(area.sourceId);
        }
      });

      mapAreas.value = [];
      highlightedAreaId.value = null;

      // Очищаем маркеры
      clearMarkers();
    };

    // Сброс выделения всех областей
    const resetHighlights = () => {
      // Сбрасываем выделение маркеров
      if (highlightedAreaId.value) {
        const prevMarker = markers.value.find(
            ({ stand }) => stand && stand.id === highlightedAreaId.value
        );
        if (prevMarker) {
          const el = prevMarker.marker.getElement();
          el.style.width = '30px';
          el.style.height = '30px';
          el.style.zIndex = '1';
          el.style.boxShadow = 'none';
        }

        // Сбрасываем выделение областей
        const prevArea = mapAreas.value.find(area => area.standId === highlightedAreaId.value);
        if (prevArea) {
          const stand = props.standData.find(s => s.id === highlightedAreaId.value);

          if (stand && stand.shape && stand.shape.style) {
            map.value.setPaintProperty(
                prevArea.id, 'fill-color', stand.shape.style.fillColor
            );
            map.value.setPaintProperty(
                prevArea.id, 'fill-opacity', stand.shape.style.fillOpacity || 0.4
            );
            map.value.setPaintProperty(
                prevArea.outlineId, 'line-width', stand.shape.style.strokeWidth || 2
            );
          }
        }

        highlightedAreaId.value = null;
      }

      // Отдаляем карту для обзора всей территории ВДНХ
      map.value.flyTo({
        center: [37.634255, 55.832910], // Центр ВДНХ
        zoom: 15,
        duration: 1000
      });
    };

    return {
      mapContainer,
      highlightStand,
      highlightArea,
      resetHighlights,
      show3DBuildings,
      toggleBuildings,
      showBuildingsToggle
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

.map-icon {
  cursor: pointer;
  transition: all 0.3s ease;
  background-color: transparent;
}

.map-icon:hover {
  transform: scale(1.1);
}

.map-icon svg {
  width: 100%;
  height: 100%;
}

.map-controls {
  position: absolute;
  top: 10px;
  right: 10px;
  z-index: 1;
  display: flex;
  flex-direction: column;
  gap: 5px;
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