<script setup>
import { ref, computed, onMounted, watch, onUnmounted } from "vue";
import { useDebounce } from "@vueuse/core";
import axios from "axios";

const searchQuery = ref("");
const debouncedSearch = useDebounce(searchQuery, 500);
const selectedGenre = ref("");
const movies = ref([]);
const featuredMovies = ref([]); // Películas destacadas para el carrusel
const genres = ref([]);
const page = ref(1);
const isLoading = ref(false);
const noResults = ref(false);
const apiKey = "e0c5e29dda61d31b996cfc46d667158d";
const apiBaseUrl = "https://api.themoviedb.org/3";
const currentSlide = ref(0);

//valores del trailer
const modalActive = ref(false);
const currentMovie = ref(null);
const movieVideos = ref([]);

const isScrolled = ref(false);

// Obtener videos de la película (tráilers)
const getMovieVideos = async (movieId) => {
  try {
    const response = await axios.get(`${apiBaseUrl}/movie/${movieId}/videos`, {
      params: {
        api_key: apiKey,
        language: "es-ES"
      }
    });
    movieVideos.value = response.data.results;
  } catch (error) {
    console.error("Error fetching videos:", error);
  }
};

const isMenuOpen = ref(false);

const toggleMenu = () => {
  isMenuOpen.value = !isMenuOpen.value;
};

// Abrir modal con los detalles
const openModal = async (movie) => {
  currentMovie.value = movie;
  await getMovieVideos(movie.id);
  modalActive.value = true;
};

// Cerrar modal
const closeModal = () => {
  modalActive.value = false;
};

const getMovies = async () => {
  isLoading.value = true;
  noResults.value = false;
  try {
    const endpoint = debouncedSearch.value
      ? `${apiBaseUrl}/search/movie`
      : `${apiBaseUrl}/discover/movie`;

    const response = await axios.get(endpoint, {
      params: {
        api_key: apiKey,
        query: debouncedSearch.value || undefined,
        page: page.value,
        with_genres: selectedGenre.value || undefined,
        language: "es-ES",
      },
    });

    if (page.value === 1) {
      movies.value = response.data.results;
    } else {
      movies.value = [...movies.value, ...response.data.results];
    }

    if (movies.value.length === 0) {
      noResults.value = true;
    }
  } catch (error) {
    console.error("Error fetching movies:", error);
  } finally {
    isLoading.value = false;
  }
};

// menu
const activeCategory = ref('home'); // Para rastrear la categoría activa

const filterByCategory = (category) => {
  activeCategory.value = category;
  searchQuery.value = ''; // Resetear búsqueda
  page.value = 1;

  switch(category) {
    case 'home':
      selectedGenre.value = '';
      getMovies();
      getFeaturedMovies(); // Recarga el carrusel principal
      break;

    case 'documentary':
      selectedGenre.value = '99'; // ID de Documentales
      getMovies();
      break;

    case 'family':
      selectedGenre.value = '10751'; // ID de Familia
      getMovies();
      break;

    case 'action':
      selectedGenre.value = '28'; // ID de Acción
      getMovies();
      break;

    case 'fantasy':
      selectedGenre.value = '14'; // ID de Fantasía
      getMovies();
      break;

    default:
      selectedGenre.value = '';
      getMovies();
  }
};
// Obtener películas destacadas para el carrusel
const getFeaturedMovies = async () => {
  try {
    const response = await axios.get(`${apiBaseUrl}/movie/upcoming`, {
      params: {
        api_key: apiKey,
        language: "es-ES",
        page: 1
      },
    });
    featuredMovies.value = response.data.results.slice(0, 6); // Tomamos las 5 más populares
  } catch (error) {
    console.error("Error fetching featured movies:", error);
  }
};

const getGenres = async () => {
  try {
    const response = await axios.get(`${apiBaseUrl}/genre/movie/list`, {
      params: { api_key: apiKey, language: "es-ES" },
    });
    genres.value = response.data.genres;
  } catch (error) {
    console.error("Error fetching genres:", error);
  }
};

// Navegación del carrusel
const nextSlide = () => {
  currentSlide.value = (currentSlide.value + 1) % featuredMovies.value.length;
};

const prevSlide = () => {
  currentSlide.value = (currentSlide.value - 1 + featuredMovies.value.length) % featuredMovies.value.length;
};

// Auto-avance del carrusel
onMounted(() => {
  getMovies();
  getFeaturedMovies();
  getGenres();
  
  setInterval(() => {
    nextSlide();
  }, 7000);
  
  window.addEventListener('scroll', handleScroll);
});

onUnmounted(() => {
  window.removeEventListener('scroll', handleScroll);
});

// Función para manejar el scroll
const handleScroll = () => {
  isScrolled.value = window.scrollY > 600;
};

watch([debouncedSearch, selectedGenre], () => {
  page.value = 1;
  getMovies();
});

const loadMoreMovies = () => {
  page.value++;
  getMovies();
};

const getPosterUrl = (posterPath) => {
  return posterPath
    ? `https://image.tmdb.org/t/p/w500${posterPath}`
    : "https://via.placeholder.com/500x750?text=No+Poster";
};

const getBackdropUrl = (backdropPath) => {
  return backdropPath
    ? `https://image.tmdb.org/t/p/w1280${backdropPath}`
    : "https://via.placeholder.com/1280x720?text=No+Backdrop";
};

const movieGenres = computed(() => {
  return movies.value.map(movie => {
    return {
      ...movie,
      genre_names: movie.genre_ids
        .map(id => genres.value.find(g => g.id === id)?.name)
        .join(", ")
    };
  });
});
</script>

<template>
  <div class="max-container">
    <!-- Barra de navegación superior -->
    <nav class="max-navbar" :class="{ 'scrolled': isScrolled }">
      <div class="navbar-left">
        <h1 class="max-logo">MOVIEXD</h1>

        <!-- Menú hamburguesa (solo móvil) -->
        <button class="menu-toggle" @click="toggleMenu">
          ☰
        </button>

         <!-- Menú normal (escritorio) -->
        <ul class="nav-links" :class="{ 'active': isMenuOpen }">
          <li @click="filterByCategory('home')">Inicio</li>
          <li @click="filterByCategory('documentary')">Documentales</li>
          <li @click="filterByCategory('family')">Niños y Familia</li>
          <li @click="filterByCategory('action')">Acción</li>
          <li @click="filterByCategory('fantasy')">Fantasía</li>
        </ul>
      </div>

      <div class="search-container">
        <input 
          v-model="searchQuery" 
          placeholder="Buscar..." 
          class="max-search"
        />
        <select v-model="selectedGenre" class="max-genre-select">
          <option value="">Todos los Géneros</option>
          <option v-for="genre in genres" :key="genre.id" :value="genre.id">
            {{ genre.name }}
          </option>
        </select>
      </div>
    </nav>

    <main class="max-main">
      <!-- Carrusel destacado -->
      <section v-if="featuredMovies.length > 0 && !debouncedSearch" class="featured-carousel">
        <div class="carousel-container">
          <button class="carousel-button prev" @click="prevSlide">‹</button>
          
          <div class="carousel-slide" 
               v-for="(movie, index) in featuredMovies" 
               :key="'featured-'+movie.id"
               :class="{ active: currentSlide === index }">
            <div class="slide-background" 
                 :style="{ backgroundImage: `url(${getBackdropUrl(movie.backdrop_path)})` }"></div>
            <div class="slide-content">
              <div class="content-wrapper">
                <h2 class="slide-title">{{ movie.title }}</h2>
                <div class="slide-meta">
                  <span class="rating">⭐ {{ movie.vote_average.toFixed(1) }}</span>
                  <span class="year">{{ movie.release_date ? movie.release_date.substring(0, 4) : 'N/A' }}</span>
                  <span class="age-rating">16+</span>
                </div>
                <p class="slide-description">{{ movie.overview || 'Descripción no disponible' }}</p>
                <div class="slide-buttons">
                  <button class="play-button" @click="openModal(movie)">▶ Ver trailer</button>
                 
                </div>
              </div>
            </div>
          </div>
          
          <button class="carousel-button next" @click="nextSlide">›</button>
        </div>
        <div class="carousel-indicators">
          <span v-for="(movie, index) in featuredMovies" 
                :key="'indicator-'+movie.id"
                :class="{ active: currentSlide === index }"
                @click="currentSlide = index"></span>
        </div>
      </section>

      <!-- Mensaje de carga -->
      <div v-if="isLoading" class="loading-container">
        <div class="max-loader"></div>
        <p>Cargando catálogo...</p>
      </div>

      <!-- Mensaje si no hay resultados -->
      <div v-if="noResults" class="no-results-container">
        <h2>No encontramos lo que buscas</h2>
        <p>Intenta con otro título o género</p>
      </div>

      <!-- Sección de películas -->
      <section v-if="movies.length > 0" class="movies-section">
        <h2 class="section-title">
          {{ debouncedSearch ? 'Resultados de búsqueda' : 'Recomendadas para ti' }}
          <span v-if="selectedGenre"> - {{ genres.find(g => g.id == selectedGenre)?.name }}</span>
        </h2>
        
        <div class="max-grid">
          <div v-for="movie in movieGenres" :key="movie.id" class="max-card" @click="openModal(movie)">
            <div class="card-image-container">
              <img 
                :src="getPosterUrl(movie.poster_path)" 
                :alt="movie.title"
                class="card-image"
              />
              <div class="card-overlay">
                <div class="overlay-content">
                  <h3>{{ movie.title }}</h3>
                  <div class="movie-meta">
                    <span class="rating">⭐ {{ movie.vote_average.toFixed(1) }}</span>
                    <span class="year">{{ movie.release_date ? movie.release_date.substring(0, 4) : 'N/A' }}</span>
                  </div>
                  <p class="genres">{{ movie.genre_names }}</p> 
                </div>
              </div>
            </div>
            <div class="card-info">
              <h3 class="card-title">{{ movie.title }}</h3>
              <div class="card-meta">
                <span class="card-rating">⭐ {{ movie.vote_average.toFixed(1) }}</span>
              </div>
            </div>
          </div>
        </div>
      </section>

      <!-- Botón de cargar más -->
      <div v-if="movies.length > 0 && !isLoading" class="load-more-container">
        <button @click="loadMoreMovies" class="max-button">
          Ver más
        </button>
      </div>
    </main>
    <!-- Modal de detalles -->
    <div v-if="modalActive" class="modal-overlay" @click.self="closeModal">
  <div class="modal-content">
    <button class="modal-close" @click="closeModal">×</button>
    
    <div class="modal-grid">
      <!-- Sección del video (izquierda) -->
      <div class="video-section" v-if="movieVideos.length > 0">
        <iframe 
          width="100%" 
          height="100%"
          :src="`https://www.youtube.com/embed/${movieVideos[0].key}`"
          frameborder="0"
          allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
          allowfullscreen>
        </iframe>
      </div>
      <div class="video-placeholder" v-else>
        <img :src="getBackdropUrl(currentMovie.backdrop_path)" alt="Movie Backdrop" class="placeholder-backdrop"/>
        <div class="placeholder-content">
          <h3>{{ currentMovie.title }}</h3>
          <p>No hay tráiler disponible</p>
        </div>
      </div>
      
      <!-- Sección de información (derecha) -->
      <div class="info-section">
        <h2 class="movie-title">{{ currentMovie.title }}</h2>
        
        <div class="details-meta">
          <span class="rating-badge">⭐ {{ currentMovie.vote_average.toFixed(1) }}</span>
          <span class="year-badge">{{ currentMovie.release_date ? currentMovie.release_date.substring(0, 4) : 'N/A' }}</span>
          <span class="genres-badge">{{ currentMovie.genre_names }}</span>
        </div>
        
        <div class="movie-details">
          <h3>Sinopsis</h3>
          <p class="movie-overview">{{ currentMovie.overview || 'Descripción no disponible' }}</p>
          
          <div class="additional-info">
            <div v-if="currentMovie.runtime" class="info-item">
              <span>Duración:</span>
              <span>{{ Math.floor(currentMovie.runtime / 60) }}h {{ currentMovie.runtime % 60 }}m</span>
            </div>
            <div class="info-item">
              <span>Fecha de estreno:</span>
              <span>{{ currentMovie.release_date || 'N/A' }}</span>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</div>
  </div>
</template>

<style scoped>
/* Estilos generales */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
  font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif;
}

.max-container {
  background-color: #0a0a0a;
  color: #fff;
  min-height: 100vh;
}

/* Barra de navegación */
.max-navbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 15px 4%;
  background-color: rgba(10, 10, 10, 0);
  position: sticky;
  top: 0;
  z-index: 100;
}

/* Menú hamburguesa (oculto en escritorio) */
.menu-toggle {
  display: none;
  background: none;
  border: none;
  color: white;
  font-size: 1.5rem;
  cursor: pointer;
}

/* Navbar en móvil (menú vertical) */
@media (max-width: 768px) {
  .max-navbar {
    flex-direction: column;
    align-items: flex-start;
    display: none;
  }


  .menu-toggle {
    display: block;
  }
  
  .nav-links {
    display: none;
    width: 100%;
    flex-direction: column;
    background: rgba(0, 0, 0, 0.9);
    padding: 10px 0;
  }
  
  .nav-links.active {
    display: flex;
  }
  
  .search-container {
    width: 100%;
    margin-top: 10px;
  }
  
  .max-search, .max-genre-select {
    width: 100%;
    margin-bottom: 5px;
  }
 
  .carousel-container {
    margin-top: 30px;
  }
}


.max-navbar.scrolled {
  background-color: rgba(10, 10, 10, 0.575) !important;
  box-shadow: rgba(0, 0, 0, 0) 0px 8px 24px;
}

.navbar-left {
  display: flex;
  align-items: center;
  gap: 30px;
}

.max-logo {
  color: #00a8e1;
  font-size: 2rem;
  font-weight: 700;
  letter-spacing: -1px;
}

.nav-links {
  display: flex;
  list-style: none;
  gap: 20px;
}

.nav-links li {
  cursor: pointer;
  padding: 8px 12px;
  border-radius: 4px;
  transition: all 0.3s;
}


.nav-links li:hover {
  background-color: rgba(0, 168, 225, 0.2);
}

.nav-links li.active {
  color: #00a8e1;
  font-weight: 600;
  background-color: rgba(0, 168, 225, 0.1);
}

.search-container {
  display: flex;
  gap: 10px;
}

.max-search {
  padding: 8px 15px;
  border: 1px solid #333;
  border-radius: 20px;
  background-color: rgba(255, 255, 255, 0.1);
  color: white;
  width: 200px;
  font-size: 0.9rem;
  outline: none;
  transition: all 0.3s;
}

.max-search::placeholder {
  color: #fcf6f6; 
  opacity: 1; 
}

.max-search:focus {
  border-color: #ffffff;
  background-color: rgba(0, 168, 225, 0.1);
}

.max-genre-select {
  padding: 8px 30px 8px 15px; /* Más espacio a la derecha para el icono */
  border: 1px solid #333;
  border-radius: 20px;
  background-color: rgba(255, 255, 255, 0.1);
  color: white;
  font-size: 0.9rem;
  outline: none;
  cursor: pointer;
  appearance: none; /* Elimina el estilo por defecto del navegador */
  -webkit-appearance: none; /* Para Safari */
  -moz-appearance: none; /* Para Firefox */
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='12' viewBox='0 0 24 24' fill='none' stroke='%23ffffff' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'%3E%3Cpolyline points='6 9 12 15 18 9'%3E%3C/polyline%3E%3C/svg%3E");
  background-repeat: no-repeat;
  background-position: right 12px center;
  background-size: 12px;
  transition: all 0.3s;
  width: 180px; /* Ancho fijo para mejor alineación */
}

.max-genre-select:hover {
  background-color: rgba(255, 255, 255, 0.15);
  border-color: #444;
}

.max-genre-select option {
  background-color: #0a0a0a;
}

.max-genre-select:focus {
  border-color: #00a8e1;
  background-color: rgba(0, 168, 225, 0.1);
}

.max-genre-select option {
  background-color: #1a1a1a;
  color: white;
  padding: 8px;
}

.max-genre-select option:hover {
  background-color: #00a8e1;
}

.max-main {
  padding: 0;
}

/* Carrusel destacado */
.featured-carousel {
  margin-top: -82px; 
  position: relative;
  height: 100vh;  
  overflow: hidden;
  margin-bottom: 40px;
}

.carousel-container {
  position: relative;
  height: 100%;
}

.carousel-slide {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  opacity: 0;
  transition: opacity 1s ease-in-out;
}

.carousel-slide.active {
  opacity: 1;
}

.slide-background {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-size: cover;
  background-position: center;
  z-index: 1;
}

.slide-background::after {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: linear-gradient(to right, rgba(10, 10, 10, 0.8) 0%, rgba(10, 10, 10, 0.4) 50%, rgba(10, 10, 10, 0.1) 100%);
}

.slide-content {
  position: relative;
  z-index: 2;
  height: 100%;
  display: flex;
  align-items: center;
  padding: 0 4%;
}

.content-wrapper {
  max-width: 40%;
}

.slide-title {
  font-size: 2.5rem;
  margin-bottom: 15px;
  font-weight: 700;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.5);
}

.slide-meta {
  display: flex;
  align-items: center;
  gap: 15px;
  margin-bottom: 20px;
  font-size: 1rem;
}

.rating {
  color: #ffffff;
  font-weight: bold;
}

.year, .age-rating {
  padding: 2px 8px;
  background-color: rgba(255, 255, 255, 0.2);
  border-radius: 4px;
  color: #ffffff;
}

.age-rating {
  border: 1px solid #fff;
}

.slide-description {
  font-size: 1rem;
  line-height: 1.5;
  margin-bottom: 25px;
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.5);
  display: -webkit-box;
  -webkit-line-clamp: 3;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

.slide-buttons {
  display: flex;
  gap: 15px;
}

.play-button, .info-button {
  padding: 10px 25px;
  border: none;
  border-radius: 4px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s;
}

.play-button {
  background-color: #00a8e1;
  color: white;
}

.play-button:hover {
  background-color: #008cbf;
}

.info-button {
  background-color: rgba(255, 255, 255, 0.2);
  color: white;
}

.info-button:hover {
  background-color: rgba(255, 255, 255, 0.3);
}

.carousel-button {
  position: absolute;
  top: 50%;
  transform: translateY(-50%);
  background-color: rgba(0, 0, 0, 0);
  color: rgb(196, 186, 186);
  border: none;
  width: 50px;
  height: 50px;
  border-radius: 50%;
  font-size: 1.5rem;
  cursor: pointer;
  z-index: 3;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.3s;
}

.carousel-button:hover { 
  color: rgb(255, 255, 255);
}

.prev {
  left: 20px;
}

.next {
  right: 20px;
}

.carousel-indicators {
  position: absolute;
  bottom: 20px;
  left: 0;
  right: 0;
  display: flex;
  justify-content: center;
  gap: 10px;
  z-index: 3;
}

.carousel-indicators span {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  background-color: rgba(255, 255, 255, 0.5);
  cursor: pointer;
  transition: all 0.3s;
}

.carousel-indicators span.active {
  background-color: #ffffff;
  transform: scale(1.2);
}

/* Sección de películas */
.movies-section {
  padding: 0 4% 40px;
}

.section-title {
  font-size: 1.5rem;
  margin-bottom: 20px;
  font-weight: 500;
  color: #e5e5e5;
}

.max-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(180px, 1fr));
  gap: 15px;
}

.max-card {
  position: relative;
  border-radius: 4px;
  overflow: hidden;
  transition: all 0.3s ease;
  aspect-ratio: 2/3;
  cursor: pointer;
}

.max-card:hover {
  transform: scale(1.05);
  z-index: 2;
}

.card-image-container {
  position: relative;
  width: 100%;
  height: 100%;
}

.card-image {
  width: 100%;
  height: 100%;
  object-fit: cover;
  display: block;
}

.card-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: linear-gradient(to top, rgba(0, 0, 0, 0.9) 0%, rgba(0, 0, 0, 0.466) 50%, rgba(0, 0, 0, 0.438) 100%);
  opacity: 0;
  transition: opacity 0.3s ease;
  padding: 15px;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
}

.max-card:hover .card-overlay {
  opacity: 1;
}

.overlay-content h3 {
  font-size: 1rem;
  margin-bottom: 8px;
  font-weight: 600;
}

.movie-meta {
  display: flex;
  gap: 10px;
  margin-bottom: 8px;
  font-size: 0.8rem;
  color: #d2d2d2;
}

.genres {
  font-size: 0.7rem;
  color: #ffffff;
  margin-bottom: 10px;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

.card-play-button {
  background-color: #00a8e1;
  color: white;
  border: none;
  width: 40px;
  height: 40px;
  border-radius: 50%;
  font-size: 1rem;
  cursor: pointer;
  margin: 10px auto 0;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.3s;
}

.card-play-button:hover {
  background-color: #008cbf;
  transform: scale(1.1);
}

.card-info {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  padding: 10px;
  background: linear-gradient(to top, rgba(0, 0, 0, 0.8) 0%, rgba(0, 0, 0, 0) 100%);
}

.card-title {
  font-size: 0.85rem;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  margin-bottom: 5px;
}

/* Botón de cargar más */
.max-button {
  background-color: #00a8e1;
  color: white;
  border: none;
  padding: 10px 30px;
  font-size: 1rem;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.max-button:hover {
  background-color: #008cbf;
}

.load-more-container {
  text-align: center;
  margin-top: 30px;
  padding-bottom: 50px;
}

/* Estilos para carga y sin resultados */
.loading-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 300px;
  gap: 20px;
}

.max-loader {
  border: 5px solid #333;
  border-top: 5px solid #00a8e1;
  border-radius: 50%;
  width: 50px;
  height: 50px;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.no-results-container {
  text-align: center;
  padding: 100px 0;
}

.no-results-container h2 {
  font-size: 1.8rem;
  color: #00a8e1;
  margin-bottom: 15px;
}

.no-results-container p {
  font-size: 1.2rem;
  color: #a3a3a3;
}
/* MODAL XD */
/* Estilos del modal */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.9);
  z-index: 1000;
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 20px;
}

.modal-content {
  background-color: #181818;
  border-radius: 8px;
  max-width: 900px;
  width: 100%;
  max-height: 80vh;
  overflow: hidden;
  position: relative;
  box-shadow: 0 10px 25px rgba(0, 0, 0, 0.5);
}

.modal-close {
  position: absolute;
  top: 15px;
  right: 15px;
  background: rgba(255, 255, 255, 0.2);
  border: none;
  color: white;
  width: 30px;
  height: 30px;
  border-radius: 50%;
  font-size: 1.2rem;
  cursor: pointer;
  z-index: 10;
}

.video-container {
  position: relative;
  padding-bottom: 56.25%; /* 16:9 aspect ratio */
  height: 0;
  background: #000;
}

.video-section iframe {
  width: 100%;
  height: 100%;
  min-height: 300px;
}

.video-container iframe {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
}

.modal-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  height: 100%;
}

.video-section, .video-placeholder {
  position: relative;
  height: 100%;
  min-height: 300px;
  background: #000;
}

.video-placeholder {
  position: relative;
  height: 100%;
  min-height: 300px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #000;
}

.placeholder-backdrop {
  width: 100%;
  height: 100%;
  object-fit: cover;
  opacity: 0.4;
}

.placeholder-content {
  position: absolute;
  text-align: center;
  padding: 20px;
  color: white;
}

.placeholder-content h3 {
  font-size: 1.5rem;
  margin-bottom: 10px;
}

.video-placeholder img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  opacity: 0.5;
}

.video-placeholder p {
  position: absolute;
  font-size: 1.2rem;
}

.video-placeholder p {
  position: absolute;
  font-size: 1.2rem;
}

.info-section {
  padding: 30px;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
}

.movie-overview {
  margin: 20px 0;
  line-height: 1.6;
  color: #d2d2d2;
}
.movie-details {
  padding: 20px;
}

.details-meta {
  display: flex;
  gap: 15px;
  margin: 10px 0;
  color: #aaa;
  font-size: 0.9rem;
}

.movie-overview {
  margin: 20px 0;
  line-height: 1.5;
}

.modal-buttons {
  display: flex;
  gap: 15px;
  margin-top: 20px;
}

.modal-button {
  padding: 10px 20px;
  border-radius: 4px;
  border: none;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s;
}

.modal-button.play {
  background-color: #00a8e1;
  color: white;
}

.modal-button.secondary {
  background-color: rgba(255, 255, 255, 0.2);
  color: white;
}

/* Responsive */
@media (max-width: 768px) {
  .modal-content {
    max-width: 95%;
  }
  
  .modal-buttons {
    flex-direction: column;
  }
}

/* Responsive */
@media (max-width: 1024px) {
  .content-wrapper {
    max-width: 60%;
  }
}

@media (max-width: 768px) {
  .max-navbar {
    flex-direction: column;
    gap: 15px;
    padding: 15px 20px;
  }
  
  .navbar-left {
    flex-direction: column;
    gap: 15px;
    width: 100%;
  }
  
  .nav-links {
    overflow-x: auto;
    width: 100%;
    padding-bottom: 5px;
  }
  
  .search-container {
    width: 100%;
  }
  
  .max-search {
    width: 100%;
  }
  
  .featured-carousel {
    height: 60vh;
  }
  
  .content-wrapper {
    max-width: 80%;
  }
  
  .slide-title {
    font-size: 1.8rem;
  }
  
  .max-grid {
    grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
  }
}

@media (max-width: 480px) {
  .featured-carousel {
    height: 50vh;
  }
  
  .content-wrapper {
    max-width: 90%;
  }
  
  .slide-title {
    font-size: 1.5rem;
  }
  
  .slide-description {
    font-size: 0.9rem;
    -webkit-line-clamp: 2;
  }
  
  .max-grid {
    grid-template-columns: repeat(auto-fill, minmax(120px, 1fr));
  }
}
</style>