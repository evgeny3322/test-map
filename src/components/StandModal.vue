<!-- components/StandModal.vue -->
<template>
  <div class="modal-overlay" @click.self="closeModal">
    <div class="modal-content" :style="{ borderTop: `5px solid ${stand.highlightColor || getStandColor()}` }">
      <div class="modal-header">
        <h2>{{ stand.name }}</h2>
        <button class="close-button" @click="closeModal">×</button>
      </div>

      <div class="modal-body">
        <div class="stand-info">
          <div class="stand-logo">
            <div v-if="stand.logoSvg" v-html="stand.logoSvg" class="stand-logo-svg"></div>
            <img v-else :src="stand.logo" :alt="stand.name" @error="handleImageError" class="stand-logo-img" />
          </div>
          <div class="stand-description">
            <p>{{ stand.description }}</p>
            <div v-if="getLocationInfo()" class="location-info">
              <strong>Местоположение:</strong> {{ getLocationInfo() }}
            </div>

            <!-- Добавлены сведения о родительской зоне -->
            <div v-if="parentArea" class="parent-area-info">
              <strong>Расположен в зоне:</strong> {{ parentArea.name }}
              <div v-if="grandParentArea" class="grand-parent-area-info">
                <strong>Территория:</strong> {{ grandParentArea.name }}
              </div>
            </div>
            
            <!-- Полная иерархия родительских зон -->
            <div v-if="parentHierarchy && parentHierarchy.length > 0" class="parent-hierarchy">
              <strong>Иерархия зон:</strong>
              <ul class="hierarchy-list">
                <li v-for="(zone, index) in parentHierarchy" :key="zone.id">
                  {{ zone.name }}
                  <span v-if="index < parentHierarchy.length - 1" class="hierarchy-separator">→</span>
                </li>
              </ul>
            </div>
          </div>
        </div>

        <div class="area-info" v-if="stand.shape">
          <h3>Информация о павильоне</h3>
          <div class="area-shape-info">
            <div><strong>Тип:</strong> {{ getShapeType() }}</div>
            <div class="area-size" v-if="getAreaSize()">
              <strong>Площадь:</strong> {{ getAreaSize() }} м²
            </div>
          </div>
        </div>

        <div class="discount-section">
          <h3>Специальные предложения и скидки</h3>
          <p>Посетите наш каталог скидок, чтобы узнать о специальных предложениях от {{ stand.name }}.</p>

          <div class="action-buttons">
            <button
                class="catalog-button"
                @click="goToCatalog(stand.discountCatalogUrl)"
                :style="{ backgroundColor: stand.highlightColor || getStandColor() }"
            >
              Перейти в каталог скидок
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed } from 'vue';

export default {
  name: 'StandModal',
  props: {
    stand: {
      type: Object,
      required: true
    },
    parentArea: {
      type: Object,
      default: null
    },
    parentHierarchy: {
      type: Array,
      default: () => []
    }
  },
  emits: ['close', 'goToCatalog'],
  setup(props, { emit }) {
    const imageError = ref(false);

    // Получение родительской зоны для промежуточной зоны (бабушка для стенда)
    const grandParentArea = computed(() => {
      if (!props.parentArea || !props.parentArea.parentAreaId) return null;
      
      // Если передана иерархия, используем ее
      if (props.parentHierarchy && props.parentHierarchy.length > 1) {
        return props.parentHierarchy[1]; // Второй элемент - бабушка
      }
      
      // Резервный вариант - проверка известных ID
      if (props.parentArea.parentAreaId === 'exhibition-complex') {
        return {
          id: 'exhibition-complex',
          name: 'Выставочный комплекс ВДНХ'
        };
      }
      return null;
    });

    // Обработка ошибки загрузки изображения
    const handleImageError = () => {
      imageError.value = true;
    };

    // Закрытие модального окна
    const closeModal = () => {
      emit('close');
    };

    // Переход в каталог скидок
    const goToCatalog = (catalogUrl) => {
      emit('goToCatalog', catalogUrl);
      closeModal();
    };

    // Получение типа фигуры на понятном языке
    const getShapeType = () => {
      if (!props.stand.shape) return '';

      const shapeTypeMap = {
        'rectangle': 'Прямоугольник',
        'polygon': 'Многоугольник',
        'circle': 'Круг',
        'semicircle': 'Полукруг'
      };

      return shapeTypeMap[props.stand.shape.type] || props.stand.shape.type;
    };

    // Получение цвета стенда из формы, если не указан highlightColor
    const getStandColor = () => {
      if (props.stand.highlightColor) return props.stand.highlightColor;
      if (props.stand.shape && props.stand.shape.style && props.stand.shape.style.fillColor) {
        return props.stand.shape.style.fillColor;
      }
      return '#333333';
    };

    // Получение информации о местоположении
    const getLocationInfo = () => {
      // Координаты ВДНХ
      if (props.stand.shape) {
        // Для павильонов ВДНХ просто возвращаем название
        return 'ВДНХ, Москва';
      }

      // Для точек с координатами
      if (props.stand.location) {
        return `${props.stand.location.lat.toFixed(6)}, ${props.stand.location.lng.toFixed(6)}`;
      }

      return null;
    };

    // Расчет примерной площади фигуры
    const getAreaSize = () => {
      if (!props.stand.shape) return null;

      switch (props.stand.shape.type) {
        case 'rectangle':
          // Для прямоугольника вычисляем площадь
          const [sw, ne] = props.stand.shape.coordinates;
          // Вычисляем расстояние в метрах
          const width = calculateDistance(
              [sw[0], sw[1]],
              [ne[0], sw[1]]
          );
          const height = calculateDistance(
              [sw[0], sw[1]],
              [sw[0], ne[1]]
          );
          return Math.round(width * height);

        case 'circle':
          // Для круга вычисляем площадь πr²
          return Math.round(Math.PI * Math.pow(props.stand.shape.radius, 2));

        case 'semicircle':
          // Для полукруга вычисляем площадь πr²/2
          return Math.round(Math.PI * Math.pow(props.stand.shape.radius, 2) / 2);

        case 'polygon':
          // Для полигона примерная площадь
          // Это упрощенный расчет для нашего случая
          return Math.round(calculatePolygonArea(props.stand.shape.coordinates));

        default:
          return null;
      }
    };

    // Вычисление расстояния между двумя точками в метрах
    const calculateDistance = (point1, point2) => {
      const [lng1, lat1] = point1;
      const [lng2, lat2] = point2;

      // Радиус Земли в метрах
      const R = 6371000;

      // Перевод в радианы
      const lat1Rad = lat1 * Math.PI / 180;
      const lat2Rad = lat2 * Math.PI / 180;
      const deltaLat = (lat2 - lat1) * Math.PI / 180;
      const deltaLng = (lng2 - lng1) * Math.PI / 180;

      // Формула гаверсинуса
      const a = Math.sin(deltaLat / 2) * Math.sin(deltaLat / 2) +
          Math.cos(lat1Rad) * Math.cos(lat2Rad) *
          Math.sin(deltaLng / 2) * Math.sin(deltaLng / 2);
      const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));

      // Расстояние в метрах
      return R * c;
    };

    // Приблизительный расчет площади многоугольника
    const calculatePolygonArea = (coordinates) => {
      // Используем формулу Гаусса для расчета площади многоугольника
      let area = 0;

      for (let i = 0; i < coordinates.length; i++) {
        const j = (i + 1) % coordinates.length;
        const [lng1, lat1] = coordinates[i];
        const [lng2, lat2] = coordinates[j];

        // Переводим координаты в метры относительно экватора и нулевого меридиана
        // Это упрощенная версия, для точности нужно учитывать кривизну Земли
        const x1 = lng1 * 111320 * Math.cos(lat1 * Math.PI / 180);
        const y1 = lat1 * 111320;
        const x2 = lng2 * 111320 * Math.cos(lat2 * Math.PI / 180);
        const y2 = lat2 * 111320;

        area += (x1 * y2 - x2 * y1);
      }

      // Берем абсолютное значение и делим на 2
      return Math.abs(area) / 2;
    };

    return {
      imageError,
      grandParentArea,
      handleImageError,
      closeModal,
      goToCatalog,
      getShapeType,
      getStandColor,
      getLocationInfo,
      getAreaSize
    };
  }
};
</script>

<style>
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.modal-content {
  background-color: white;
  width: 90%;
  max-width: 600px;
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.15);
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 15px 20px;
  background-color: #f8f8f8;
}

.modal-header h2 {
  margin: 0;
  font-size: 1.5rem;
}

.close-button {
  background: none;
  border: none;
  font-size: 1.8rem;
  cursor: pointer;
  color: #555;
}

.modal-body {
  padding: 20px;
}

.stand-info {
  display: flex;
  gap: 20px;
  margin-bottom: 25px;
}

.stand-logo {
  flex: 0 0 120px;
  height: 120px;
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: #f8f8f8;
  border-radius: 6px;
  overflow: hidden;
}

.stand-logo-img {
  max-width: 100%;
  max-height: 100%;
  object-fit: contain;
  transition: transform 0.3s ease;
}

.stand-logo-img:hover {
  transform: scale(1.05);
}

.stand-logo-svg {
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: transform 0.3s ease;
}

.stand-logo-svg:hover {
  transform: scale(1.05);
}

.stand-logo-svg svg {
  width: 100%;
  height: 100%;
}

.stand-description {
  flex: 1;
}

.location-info {
  margin-top: 10px;
  font-size: 0.9rem;
  color: #666;
}

.area-info {
  background-color: #f7f7f7;
  padding: 15px;
  border-radius: 6px;
  margin-bottom: 20px;
}

.area-info h3 {
  margin-top: 0;
  margin-bottom: 10px;
  color: #333;
  font-size: 1.1rem;
}

.area-shape-info {
  display: flex;
  flex-wrap: wrap;
  gap: 15px;
}

.discount-section {
  background-color: #f9f9f9;
  padding: 15px;
  border-radius: 6px;
}

.discount-section h3 {
  margin-top: 0;
  color: #333;
}

.action-buttons {
  margin-top: 20px;
  display: flex;
  justify-content: center;
}

.catalog-button {
  padding: 10px 20px;
  border: none;
  border-radius: 4px;
  color: white;
  font-weight: bold;
  cursor: pointer;
  transition: all 0.3s ease;
}

.catalog-button:hover {
  opacity: 0.9;
  transform: translateY(-2px);
}

.parent-area-info {
  margin-top: 10px;
  font-size: 0.9rem;
  color: #666;
}

.grand-parent-area-info {
  margin-top: 5px;
  font-size: 0.8rem;
  color: #999;
}

.parent-hierarchy {
  margin-top: 10px;
  padding: 5px 10px;
  background-color: #f5f5f5;
  border-radius: 4px;
  font-size: 0.9rem;
}

.hierarchy-list {
  margin: 5px 0 0;
  padding-left: 20px;
  list-style-type: none;
  display: flex;
  flex-wrap: wrap;
}

.hierarchy-list li {
  margin-right: 5px;
  color: #666;
}

.hierarchy-separator {
  margin: 0 5px;
  color: #999;
}
</style>