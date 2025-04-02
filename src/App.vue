<template>
  <div class="app-container">
    <h1>Интерактивная карта стендов</h1>
    <InteractiveMap
        :standData="stands"
        @openStandDetails="openStandDetails"
    />

    <StandModal
        v-if="selectedStand"
        :stand="selectedStand"
        @close="closeStandDetails"
        @goToCatalog="goToCatalog"
    />
  </div>
</template>

<script>
import { ref } from 'vue';
import InteractiveMap from './components/InteractiveMap.vue';
import StandModal from './components/StandModal.vue';

export default {
  name: 'App',
  components: {
    InteractiveMap,
    StandModal
  },
  setup() {
    const selectedStand = ref(null);

    // Пример данных стендов (в реальном проекте эти данные могут приходить с API)
    const stands = ref([
      {
        id: 1,
        name: 'Стенд ACME Inc.',
        logo: '/acme-logo.svg',
        description: 'Инновационные решения для бизнеса от ACME Inc. Представляем новейшие технологические решения для автоматизации бизнес-процессов.',
        location: {
          lng: 37.618423,
          lat: 55.751244
        },
        discountCatalogUrl: '/catalog/acme',
        highlightColor: '#FF5733'
      },
      {
        id: 2,
        name: 'TechGlobal',
        logo: '/techglobal-logo.svg',
        description: 'Передовые технологии для вашего бизнеса. Международная компания, специализирующаяся на IT-решениях для предприятий любого масштаба.',
        location: {
          lng: 37.620423,
          lat: 55.753244
        },
        discountCatalogUrl: '/catalog/techglobal',
        highlightColor: '#3385FF'
      },
      {
        id: 3,
        name: 'FutureVision Labs',
        logo: '/futurevision-logo.svg',
        description: 'Взгляд в будущее с FutureVision Labs. Разработка инновационных решений в области компьютерного зрения и искусственного интеллекта.',
        location: {
          lng: 37.615423,
          lat: 55.749244
        },
        discountCatalogUrl: '/catalog/futurevision',
        highlightColor: '#33FF57'
      },
      {
        id: 4,
        name: 'SmartDev Solutions',
        logo: '/smartdev-logo.svg',
        description: 'Разработка умных приложений для бизнеса и образования. Наши решения помогают оптимизировать процессы и увеличивать эффективность.',
        location: {
          lng: 37.617423,
          lat: 55.748244
        },
        discountCatalogUrl: '/catalog/smartdev',
        highlightColor: '#9C27B0'
      },
      {
        id: 5,
        name: 'EcoInnovate',
        logo: '/ecoinnovate-logo.svg',
        description: 'Экологичные инновации для современного мира. Мы разрабатываем технологии, которые помогают сохранять природные ресурсы.',
        location: {
          lng: 37.621423,
          lat: 55.750244
        },
        discountCatalogUrl: '/catalog/ecoinnovate',
        highlightColor: '#4CAF50'
      }
    ]);

    // Открыть информацию о выбранном стенде
    const openStandDetails = (stand) => {
      selectedStand.value = stand;
    };

    // Закрыть модальное окно
    const closeStandDetails = () => {
      selectedStand.value = null;
    };

    // Переход в каталог скидок
    const goToCatalog = (catalogUrl) => {
      console.log('Переход в каталог скидок:', catalogUrl);
      // Здесь можно реализовать навигацию на страницу каталога
      window.location.href = catalogUrl;
      // Или, если используется Vue Router:
      // router.push(catalogUrl);
    };

    return {
      stands,
      selectedStand,
      openStandDetails,
      closeStandDetails,
      goToCatalog
    };
  }
};
</script>

<style>
.app-container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
  font-family: Arial, sans-serif;
}

h1 {
  text-align: center;
  margin-bottom: 20px;
}
</style>