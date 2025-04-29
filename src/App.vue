<template>
  <div class="app-container">
    <h1>Интерактивная карта ВДНХ</h1>
    <InteractiveMap
        :standData="stands"
        :mapMarkers="mapMarkers"
        @openStandDetails="openStandDetails"
    />

    <!-- Модальное окно с информацией о стенде -->
    <StandModal
        v-if="selectedStand"
        :stand="selectedStand"
        :parentArea="getParentArea(selectedStand)"
        @close="closeStandDetails"
        @goToCatalog="goToCatalog"
    />
  </div>
</template>

<script>
import { ref, computed } from 'vue';
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
      food: `<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 200 200">
        <circle cx="100" cy="100" r="90" fill="#E91E63" />
        <circle cx="100" cy="100" r="80" fill="#FFFFFF" />
        <path d="M70 70 L70 130 L130 130 L130 70 Z" fill="#E91E63" opacity="0.2" />
        <path d="M80 65 L80 75 L120 75 L120 65 Z" fill="#E91E63" />
        <path d="M75 85 L125 85 M75 95 L125 95 M75 105 L125 105 M75 115 L125 115" stroke="#E91E63" stroke-width="4" />
      </svg>`,

      entertainment: `<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 200 200">
        <circle cx="100" cy="100" r="90" fill="#FF9800" />
        <circle cx="100" cy="100" r="80" fill="#FFFFFF" />
        <path d="M60 100 A40 40 0 1 0 140 100 A40 40 0 1 0 60 100" fill="none" stroke="#FF9800" stroke-width="3" />
        <circle cx="100" cy="60" r="10" fill="#FF9800" />
        <circle cx="140" cy="100" r="10" fill="#FF9800" />
        <circle cx="100" cy="140" r="10" fill="#FF9800" />
        <circle cx="60" cy="100" r="10" fill="#FF9800" />
        <circle cx="100" cy="100" r="15" fill="#FF9800" opacity="0.6" />
      </svg>`,

      museum: `<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 200 200">
        <circle cx="100" cy="100" r="90" fill="#3F51B5" />
        <circle cx="100" cy="100" r="80" fill="#FFFFFF" />
        <path d="M50 140 L50 90 L100 60 L150 90 L150 140 Z" fill="none" stroke="#3F51B5" stroke-width="3" />
        <path d="M60 140 L60 100 L100 75 L140 100 L140 140" fill="none" stroke="#3F51B5" stroke-width="2" />
        <path d="M70 140 L70 110 M100 140 L100 110 M130 140 L130 110" stroke="#3F51B5" stroke-width="4" />
        <path d="M50 140 L150 140" stroke="#3F51B5" stroke-width="6" />
      </svg>`,
      
      exhibition: `<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 200 200">
        <circle cx="100" cy="100" r="90" fill="#8BC34A" />
        <circle cx="100" cy="100" r="80" fill="#FFFFFF" />
        <rect x="50" y="70" width="100" height="60" rx="5" fill="#8BC34A" opacity="0.2" />
        <path d="M60 80 L60 120 M90 80 L90 120 M120 80 L120 120 M150 80 L150 120" stroke="#8BC34A" stroke-width="3" />
        <path d="M50 70 L150 70 M50 130 L150 130" stroke="#8BC34A" stroke-width="5" />
      </svg>`
    };

    // Большая родительская зона "Выставочный комплекс"
    const exhibitionArea = {
      id: 'exhibition-complex',
      name: 'Выставочный комплекс ВДНХ',
      logoSvg: logosSvg.exhibition,
      description: 'Главный выставочный комплекс, объединяющий несколько павильонов и стендов с различными экспозициями. Ежегодно здесь проводятся международные выставки и форумы.',
      shape: {
        type: 'polygon',
        coordinates: [
          [37.6230, 55.8310],
          [37.6260, 55.8330],
          [37.6300, 55.8320],
          [37.6320, 55.8300],
          [37.6310, 55.8280],
          [37.6250, 55.8270],
          [37.6220, 55.8290]
        ],
        style: {
          fillColor: '#8BC34A',
          strokeColor: '#8BC34A',
          fillOpacity: 0.3,
          strokeWidth: 2
        }
      },
      zIndex: 1, // Нижний слой
      isClickable: false, // Не кликабельный, только для отображения общей зоны
      discountCatalogUrl: '/catalog/exhibition',
      highlightColor: '#8BC34A'
    };

    // Стенды, расположенные внутри выставочного комплекса
    const exhibitionStands = [
      {
        id: 'tech-stand-1',
        name: 'Павильон технологий',
        logoSvg: logosSvg.museum,
        parentAreaId: 'exhibition-complex', // Ссылка на родительскую зону
        description: 'Современный павильон, демонстрирующий последние достижения в области технологий и инноваций.',
        shape: {
          type: 'rectangle',
          coordinates: [
            [37.6240, 55.8290],
            [37.6260, 55.8310]
          ],
          style: {
            fillColor: '#3F51B5',
            strokeColor: '#3F51B5',
            fillOpacity: 0.6,
            strokeWidth: 2
          }
        },
        zIndex: 2, // Слой выше родительской зоны
        isClickable: true,
        discountCatalogUrl: '/catalog/tech-stand',
        highlightColor: '#3F51B5'
      },
      {
        id: 'food-stand-1',
        name: 'Гастрономический павильон',
        logoSvg: logosSvg.food,
        parentAreaId: 'exhibition-complex',
        description: 'Павильон с кулинарными мастер-классами и дегустацией блюд национальной кухни.',
        shape: {
          type: 'circle',
          coordinates: [37.6280, 55.8300],
          radius: 30, // метры
          style: {
            fillColor: '#E91E63',
            strokeColor: '#E91E63',
            fillOpacity: 0.5,
            strokeWidth: 2
          }
        },
        zIndex: 2,
        isClickable: true,
        discountCatalogUrl: '/catalog/food-stand',
        highlightColor: '#E91E63'
      },
      {
        id: 'entertainment-stand-1',
        name: 'Развлекательная зона',
        logoSvg: logosSvg.entertainment,
        parentAreaId: 'exhibition-complex',
        description: 'Зона развлечений с интерактивными аттракционами и играми для всей семьи.',
        shape: {
          type: 'polygon',
          coordinates: [
            [37.6270, 55.8275],
            [37.6290, 55.8280],
            [37.6295, 55.8290],
            [37.6280, 55.8295],
            [37.6265, 55.8285]
          ],
          style: {
            fillColor: '#FF9800',
            strokeColor: '#FF9800',
            fillOpacity: 0.5,
            strokeWidth: 2
          }
        },
        zIndex: 2,
        isClickable: true,
        discountCatalogUrl: '/catalog/entertainment-stand',
        highlightColor: '#FF9800'
      }
    ];

    // Основные локации на ВДНХ
    const vdnhAreas = [
      // Добавляем главную зону выставочного комплекса
      exhibitionArea,
      // Добавляем стенды внутри выставочного комплекса
      ...exhibitionStands,
      
      // 1. Парк аттракционов - полигон сложной формы
      {
        id: 'amusement-park',
        name: 'Парк аттракционов "Город развлечений"',
        logoSvg: logosSvg.entertainment,
        description: 'Крупнейший парк развлечений на территории ВДНХ с разнообразными аттракционами для детей и взрослых. Здесь находятся карусели, американские горки, колесо обозрения и множество других экстремальных и семейных аттракционов.',
        shape: {
          type: 'polygon',
          coordinates: [
            [37.626500, 55.832800],
            [37.628500, 55.833200],
            [37.629300, 55.832500],
            [37.628800, 55.831800],
            [37.627500, 55.831500],
            [37.626200, 55.832000]
          ],
          style: {
            fillColor: '#FF9800',
            strokeColor: '#FF9800',
            fillOpacity: 0.4,
            strokeWidth: 2
          }
        },
        zIndex: 1,
        isClickable: true,
        discountCatalogUrl: '/catalog/amusement',
        highlightColor: '#FF9800'
      },

      // 2. Музей космонавтики - прямоугольник
      {
        id: 'space-museum',
        name: 'Музей космонавтики',
        logoSvg: logosSvg.museum,
        description: 'Один из крупнейших научно-технических музеев мира, посвященный истории освоения космоса. В экспозиции представлены образцы космической техники, личные вещи космонавтов, архивные документы и многое другое.',
        shape: {
          type: 'polygon',
          coordinates: [
            [37.6202045922669, 55.83581591923999],
            [37.621688445748674, 55.834924897441],
            [37.622151190912604, 55.835164638533],
            [37.62068224732761, 55.83606248103746],
            [37.6202023764796, 55.835828842959785]
          ],
          style: {
            fillColor: '#3F51B5',
            strokeColor: '#3F51B5',
            fillOpacity: 0.6,
            strokeWidth: 3
          }
        },
        zIndex: 1,
        isClickable: true,
        discountCatalogUrl: '/catalog/space-museum',
        highlightColor: '#3F51B5'
      },

      // 3. Ресторанный комплекс - круг
      {
        id: 'dining-complex',
        name: 'Ресторанный комплекс "Вкусная площадь"',
        logoSvg: logosSvg.food,
        description: 'Современный гастрономический комплекс, объединяющий рестораны различных кухонь мира, кафе, фуд-корты и кондитерские. Отличное место для отдыха и дегустации блюд национальной и международной кухни.',
        shape: {
          type: 'circle',
          coordinates: [37.631000, 55.830500],
          radius: 60, // метры
          style: {
            fillColor: '#E91E63',
            strokeColor: '#E91E63',
            fillOpacity: 0.4,
            strokeWidth: 2
          }
        },
        zIndex: 1,
        isClickable: true,
        discountCatalogUrl: '/catalog/dining',
        highlightColor: '#E91E63'
      }
    ];

    // Информационные пиктограммы
    const mapIcons = [
      {
        id: 'info-kiosk-1',
        name: 'Информационный киоск №1',
        coordinates: [37.628200, 55.831200],
        icon: `<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 200 200">
          <circle cx="100" cy="100" r="90" fill="#2196F3" />
          <circle cx="100" cy="100" r="80" fill="#FFFFFF" />
          <circle cx="100" cy="80" r="12" fill="#2196F3" />
          <rect x="90" y="100" width="20" height="40" rx="2" fill="#2196F3" />
        </svg>`,
        size: 40
      },
      {
        id: 'toilet-1',
        name: 'Туалет',
        coordinates: [37.629800, 55.832300],
        icon: `<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 200 200">
          <circle cx="100" cy="100" r="90" fill="#607D8B" />
          <circle cx="100" cy="100" r="80" fill="#FFFFFF" />
          <circle cx="80" cy="80" r="15" fill="#607D8B" />
          <circle cx="120" cy="80" r="15" fill="#607D8B" />
          <path d="M60 110 L80 170 L120 170 L140 110" fill="none" stroke="#607D8B" stroke-width="6" />
        </svg>`,
        size: 40
      },
      {
        id: 'parking-1',
        name: 'Парковка',
        coordinates: [37.633800, 55.830800],
        icon: `<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 200 200">
          <circle cx="100" cy="100" r="90" fill="#4CAF50" />
          <circle cx="100" cy="100" r="80" fill="#FFFFFF" />
          <rect x="70" y="60" width="60" height="80" rx="5" fill="#4CAF50" />
          <path d="M80 80 L120 80 M80 100 L120 100 M80 120 L120 120" stroke="#FFFFFF" stroke-width="5" />
        </svg>`,
        size: 40
      }
    ];

    // Устанавливаем данные в реактивные переменные
    const stands = ref(vdnhAreas);
    const mapMarkers = ref(mapIcons);

    // Получение родительской зоны для стенда
    const getParentArea = (stand) => {
      if (!stand || !stand.parentAreaId) return null;
      return vdnhAreas.find(area => area.id === stand.parentAreaId);
    };

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
      mapMarkers,
      selectedStand,
      getParentArea,
      openStandDetails,
      closeStandDetails,
      goToCatalog
    };
  }
};
</script>

<style>
.app-container {
  max-width: 1500px;
  margin: 0 auto;
  padding: 20px;
  font-family: Arial, sans-serif;
}

h1 {
  text-align: center;
  margin-bottom: 20px;
  color: #333;
}
</style>