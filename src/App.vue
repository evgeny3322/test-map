<template>
  <div class="app-container">
    <h1>Интерактивная карта стендов</h1>
    <InteractiveMap
        :standData="stands"
        @openStandDetails="openStandDetails"
    />

    <!-- Модальное окно с информацией о стенде -->
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

    // SVG-коды логотипов
    const logosSvg = {
      acme: `<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 200 200">
        <circle cx="100" cy="100" r="90" fill="#FF5733" />
        <circle cx="100" cy="100" r="80" fill="#FFFFFF" />
        <path d="M40 120 L60 70 L70 70 L90 120 L80 120 L75 105 L55 105 L50 120 Z M58 95 L72 95 L65 75 Z" fill="#FF5733" />
        <path d="M95 120 L95 70 L125 70 Q135 70 140 75 Q145 80 145 90 Q145 100 140 105 Q135 110 125 110 L105 110 L105 120 Z M105 100 L125 100 Q128 100 130 98 Q132 96 132 90 Q132 84 130 82 Q128 80 125 80 L105 80 Z" fill="#FF5733" />
        <path d="M150 120 L150 70 L180 70 Q190 70 190 80 L190 85 Q190 93 180 93 Q190 93 190 100 L190 110 Q190 120 180 120 Z M160 90 L180 90 Q183 90 183 87 L183 83 Q183 80 180 80 L160 80 Z M160 110 L180 110 Q183 110 183 107 L183 103 Q183 100 180 100 L160 100 Z" fill="#FF5733" />
        <path d="M25 120 L25 70 L60 70 L60 80 L35 80 L35 90 L55 90 L55 100 L35 100 L35 110 L60 110 L60 120 Z" fill="#FF5733" />
        <circle cx="100" cy="100" r="90" fill="none" stroke="#FF5733" stroke-width="5" />
      </svg>`,

      techglobal: `<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 200 200">
        <rect x="10" y="10" width="180" height="180" rx="15" fill="#3385FF" />
        <rect x="20" y="20" width="160" height="160" rx="10" fill="#FFFFFF" />
        <circle cx="100" cy="100" r="50" fill="none" stroke="#3385FF" stroke-width="3" />
        <circle cx="100" cy="100" r="40" fill="none" stroke="#3385FF" stroke-width="2" />
        <ellipse cx="100" cy="100" rx="50" ry="20" fill="none" stroke="#3385FF" stroke-width="3" />
        <ellipse cx="100" cy="100" rx="25" ry="50" fill="none" stroke="#3385FF" stroke-width="3" transform="rotate(45, 100, 100)" />
        <ellipse cx="100" cy="100" rx="25" ry="50" fill="none" stroke="#3385FF" stroke-width="3" transform="rotate(-45, 100, 100)" />
        <circle cx="135" cy="65" r="8" fill="#3385FF" />
        <circle cx="65" cy="65" r="8" fill="#3385FF" />
        <circle cx="65" cy="135" r="8" fill="#3385FF" />
        <circle cx="135" cy="135" r="8" fill="#3385FF" />
        <circle cx="100" cy="50" r="8" fill="#3385FF" />
        <circle cx="100" cy="150" r="8" fill="#3385FF" />
        <circle cx="50" cy="100" r="8" fill="#3385FF" />
        <circle cx="150" cy="100" r="8" fill="#3385FF" />
        <path d="M65 175 L65 160 L90 160 L90 175 Z M75 175 L75 160" fill="#3385FF" />
        <path d="M95 175 L95 160 L110 160 L110 167.5 L95 167.5 L95 175 Z" fill="#3385FF" />
        <path d="M115 175 L115 160 L135 160 L135 175 Z" fill="#3385FF" />
      </svg>`,

      futurevision: `<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 200 200">
        <polygon points="100,10 170,50 170,150 100,190 30,150 30,50" fill="#33FF57" />
        <polygon points="100,25 155,58 155,142 100,175 45,142 45,58" fill="#FFFFFF" />
        <ellipse cx="100" cy="100" rx="40" ry="25" fill="none" stroke="#33FF57" stroke-width="5" />
        <circle cx="100" cy="100" r="15" fill="#33FF57" />
        <circle cx="100" cy="100" r="8" fill="#FFFFFF" />
        <line x1="30" y1="100" x2="60" y2="100" stroke="#33FF57" stroke-width="3" />
        <line x1="140" y1="100" x2="170" y2="100" stroke="#33FF57" stroke-width="3" />
        <line x1="70" y1="60" x2="70" y2="75" stroke="#33FF57" stroke-width="3" />
        <line x1="130" y1="60" x2="130" y2="75" stroke="#33FF57" stroke-width="3" />
        <line x1="70" y1="125" x2="70" y2="140" stroke="#33FF57" stroke-width="3" />
        <line x1="130" y1="125" x2="130" y2="140" stroke="#33FF57" stroke-width="3" />
        <path d="M75 160 L75 150 L125 150 L125 160 Z" fill="#33FF57" />
        <path d="M85 170 L95 150 L105 170 Z" fill="#33FF57" />
        <path d="M115 170 L125 150 L135 170 Z" fill="#33FF57" />
      </svg>`,

      smartdev: `<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 200 200">
        <rect x="10" y="10" width="180" height="180" rx="30" fill="#9C27B0" />
        <rect x="20" y="20" width="160" height="160" rx="25" fill="#FFFFFF" />
        <path d="M60 90 C60 70, 80 50, 100 50 C120 50, 140 70, 140 90 C140 110, 120 130, 100 130 C80 130, 60 110, 60 90"
              fill="none" stroke="#9C27B0" stroke-width="4" />
        <path d="M70 80 C75 70, 90 65, 100 65 C110 65, 125 70, 130 80"
              fill="none" stroke="#9C27B0" stroke-width="3" />
        <path d="M70 100 C75 110, 90 115, 100 115 C110 115, 125 110, 130 100"
              fill="none" stroke="#9C27B0" stroke-width="3" />
        <circle cx="100" cy="50" r="6" fill="#9C27B0" />
        <circle cx="140" cy="90" r="6" fill="#9C27B0" />
        <circle cx="100" cy="130" r="6" fill="#9C27B0" />
        <circle cx="60" cy="90" r="6" fill="#9C27B0" />
        <line x1="100" y1="50" x2="100" y2="30" stroke="#9C27B0" stroke-width="3" />
        <line x1="140" y1="90" x2="160" y2="90" stroke="#9C27B0" stroke-width="3" />
        <line x1="100" y1="130" x2="100" y2="150" stroke="#9C27B0" stroke-width="3" />
        <line x1="60" y1="90" x2="40" y2="90" stroke="#9C27B0" stroke-width="3" />
        <circle cx="100" cy="30" r="4" fill="#9C27B0" />
        <circle cx="160" cy="90" r="4" fill="#9C27B0" />
        <circle cx="100" cy="150" r="4" fill="#9C27B0" />
        <circle cx="40" cy="90" r="4" fill="#9C27B0" />
        <text x="90" y="175" font-family="Arial" font-size="24" font-weight="bold" fill="#9C27B0">SD</text>
      </svg>`,

      ecoinnovate: `<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 200 200">
        <circle cx="100" cy="100" r="90" fill="#4CAF50" />
        <circle cx="100" cy="100" r="80" fill="#FFFFFF" />
        <path d="M100 40 C140 40, 160 80, 150 120 C140 150, 100 160, 70 150 C40 140, 30 100, 50 70 C70 40, 100 40, 100 40 Z"
              fill="#4CAF50" opacity="0.2" />
        <path d="M100 50 C130 50, 145 80, 140 110 C130 140, 100 145, 80 135 C50 125, 45 100, 60 75 C75 50, 100 50, 100 50 Z"
              fill="#4CAF50" opacity="0.4" />
        <path d="M100 60 C120 60, 130 80, 125 100 C120 120, 100 130, 80 120 C60 110, 55 90, 70 75 C80 60, 100 60, 100 60 Z"
              fill="#4CAF50" opacity="0.6" />
        <path d="M100 130 C100 130, 95 140, 95 150 C95 160, 105 160, 105 150 C105 140, 100 130, 100 130 Z"
              fill="#4CAF50" />
        <path d="M70 80 L100 80 L130 80" fill="none" stroke="#4CAF50" stroke-width="2" />
        <path d="M60 100 L100 100 L140 100" fill="none" stroke="#4CAF50" stroke-width="2" />
        <path d="M70 120 L100 120 L130 120" fill="none" stroke="#4CAF50" stroke-width="2" />
        <path d="M100 60 L100 130" fill="none" stroke="#4CAF50" stroke-width="2" />
        <circle cx="100" cy="80" r="3" fill="#4CAF50" />
        <circle cx="100" cy="100" r="3" fill="#4CAF50" />
        <circle cx="100" cy="120" r="3" fill="#4CAF50" />
        <circle cx="70" cy="80" r="3" fill="#4CAF50" />
        <circle cx="130" cy="80" r="3" fill="#4CAF50" />
        <circle cx="60" cy="100" r="3" fill="#4CAF50" />
        <circle cx="140" cy="100" r="3" fill="#4CAF50" />
        <circle cx="70" cy="120" r="3" fill="#4CAF50" />
        <circle cx="130" cy="120" r="3" fill="#4CAF50" />
        <path d="M60 170 L60 155 L85 155 L85 160 L65 160 L65 170 Z" fill="#4CAF50" />
        <path d="M90 170 L90 155 L115 155 C120 155, 120 170, 115 170 Z" fill="#4CAF50" />
        <path d="M125 170 L125 155 L140 155 L140 170 Z M130 170 L130 155" fill="#4CAF50" />
      </svg>`
    };

    // Пример данных стендов (в реальном проекте эти данные могут приходить с API)
    const stands = ref([
      {
        id: 1,
        name: 'Стенд ACME Inc.',
        logoSvg: logosSvg.acme,
        description: 'Инновационные решения для бизнеса от ACME Inc. Представляем новейшие технологические решения для автоматизации бизнес-процессов.',
        location: {
          lng: 37.556441, // Крокус Экспо
          lat: 55.825983
        },
        discountCatalogUrl: '/catalog/acme',
        highlightColor: '#FF5733'
      },
      {
        id: 2,
        name: 'TechGlobal',
        logoSvg: logosSvg.techglobal,
        description: 'Передовые технологии для вашего бизнеса. Международная компания, специализирующаяся на IT-решениях для предприятий любого масштаба.',
        location: {
          lng: 37.537021, // Москва-Сити, Башня "Федерация"
          lat: 55.749451
        },
        discountCatalogUrl: '/catalog/techglobal',
        highlightColor: '#3385FF'
      },
      {
        id: 3,
        name: 'FutureVision Labs',
        logoSvg: logosSvg.futurevision,
        description: 'Взгляд в будущее с FutureVision Labs. Разработка инновационных решений в области компьютерного зрения и искусственного интеллекта.',
        location: {
          lng: 37.632595, // Технополис "Москва"
          lat: 55.731772
        },
        discountCatalogUrl: '/catalog/futurevision',
        highlightColor: '#33FF57'
      },
      {
        id: 4,
        name: 'SmartDev Solutions',
        logoSvg: logosSvg.smartdev,
        description: 'Разработка умных приложений для бизнеса и образования. Наши решения помогают оптимизировать процессы и увеличивать эффективность.',
        location: {
          lng: 37.403630, // Инновационный центр "Сколково"
          lat: 55.751244
        },
        discountCatalogUrl: '/catalog/smartdev',
        highlightColor: '#9C27B0'
      },
      {
        id: 5,
        name: 'EcoInnovate',
        logoSvg: logosSvg.ecoinnovate,
        description: 'Экологичные инновации для современного мира. Мы разрабатываем технологии, которые помогают сохранять природные ресурсы.',
        location: {
          lng: 37.587093, // Экспоцентр на Красной Пресне
          lat: 55.750251
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