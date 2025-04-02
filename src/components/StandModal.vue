<template>
  <div class="modal-overlay" @click.self="closeModal">
    <div class="modal-content" :style="{ borderTop: `5px solid ${stand.highlightColor || '#FF0000'}` }">
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
            <div class="stand-location">
              <strong>Координаты:</strong>
              {{ stand.location.lat.toFixed(6) }}, {{ stand.location.lng.toFixed(6) }}
              <div v-if="getLocationName(stand)" class="location-name">
                <strong>Место:</strong> {{ getLocationName(stand) }}
              </div>
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
                :style="{ backgroundColor: stand.highlightColor || '#FF0000' }"
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
import { ref } from 'vue';

export default {
  name: 'StandModal',
  props: {
    stand: {
      type: Object,
      required: true
    }
  },
  emits: ['close', 'goToCatalog'],
  setup(props, { emit }) {
    const imageError = ref(false);

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

    // Получение названия локации по координатам
    const getLocationName = (stand) => {
      const { lng, lat } = stand.location;

      // Крокус Экспо
      if (Math.abs(lng - 37.556441) < 0.001 && Math.abs(lat - 55.825983) < 0.001) {
        return 'Крокус Экспо';
      }

      // Москва-Сити, Башня "Федерация"
      if (Math.abs(lng - 37.537021) < 0.001 && Math.abs(lat - 55.749451) < 0.001) {
        return 'Москва-Сити, Башня "Федерация"';
      }

      // Технополис "Москва"
      if (Math.abs(lng - 37.632595) < 0.001 && Math.abs(lat - 55.731772) < 0.001) {
        return 'Технополис "Москва"';
      }

      // Инновационный центр "Сколково"
      if (Math.abs(lng - 37.403630) < 0.001 && Math.abs(lat - 55.751244) < 0.001) {
        return 'Инновационный центр "Сколково"';
      }

      // Экспоцентр на Красной Пресне
      if (Math.abs(lng - 37.587093) < 0.001 && Math.abs(lat - 55.750251) < 0.001) {
        return 'Экспоцентр на Красной Пресне';
      }

      return '';
    };

    return {
      imageError,
      handleImageError,
      closeModal,
      goToCatalog,
      getLocationName
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

.stand-location {
  margin-top: 10px;
  font-size: 0.9rem;
  color: #666;
}

.location-name {
  margin-top: 5px;
  color: #333;
  font-weight: 500;
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
</style>