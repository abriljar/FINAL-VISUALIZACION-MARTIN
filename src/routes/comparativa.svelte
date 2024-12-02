<!-- Script JS -->
<script>
	import * as d3 from "d3";
	import { onMount } from "svelte";

	let width = 400;
	let height = 400;
	let filtro = "todas";
	let usuarios = [];
	let songData = {}; // Datos de las canciones mapeadas por nombre
	let currentSongData = {}; // Datos actuales para el gráfico

	const energyRotationSpeeds = {
		Alta: "2s", // Rápida
		Media: "6s", // Media
		Baja: "16s", // Lenta
	};

	$: {
		if (filtro != "todas") {
			usuarios = usuarios.filter((p) => p.Usuario == filtro);
		}
	}

	onMount(async () => {
		// Cargar datos desde el archivo CSV
		const csvData = await d3.csv("src/data/Spotify.csv");
		songData = csvData.reduce((acc, song) => {
			acc[song.Nombre] = {
				Acousticness: +song.Acousticness,
				Danceability: +song.Danceability,
				Energy: song.Energy.trim(), // Normalizar energía también
				Instrumentalness: +song.Instrumentalness,
				Liveness: +song.Liveness,
				Speechiness: +song.Speechiness,
				Artista: song.Artista,
			};
			return acc;
		}, {});

		// Cargar los datos iniciales del gráfico
		if (csvData.length > 0) {
			currentSongData = Object.values(songData)[0]; // Primera canción como inicial
			createRadarChart(currentSongData);
		}
	});

	function createRadarChart(data) {
		d3.select("#radar-chart").selectAll("*").remove(); // Limpiar el gráfico previo
		const relevantKeys = [
			"Valence",
			"Instrumentalness",
			"Liveness",
			"Acousticness",
			"Danceability",
			"Speechiness",
		];
		const filteredData = relevantKeys.reduce((acc, key) => {
			acc[key] = data[key] || 0; // Asegúrate de manejar valores faltantes con un valor predeterminado
			return acc;
		}, {});
		const keys = Object.keys(filteredData); // Claves filtradas (ejes del radar)
		const values = Object.values(filteredData); // Valores para cada eje

		const radius = Math.min(width, height) / 1 - 80; // Radio máximo del gráfico
		const angleSlice = (Math.PI * 2) / keys.length; // Ángulo entre ejes
		const radialScale = d3.scaleLinear().domain([0, 1]).range([0, radius]); // Escala radial

		const svg = d3
			.select("#radar-chart")
			.attr("width", width)
			.attr("height", height)
			.append("g")
			.attr("transform", `translate(${width / 2}, ${height / 2})`); // Centrar el gráfico

		const tooltip = d3
			.select("body")
			.append("div")
			.attr("class", "tooltip")
			.style("position", "absolute")
			.style("background", "rgba(0, 0, 0, 0.7)")
			.style("color", "white")
			.style("padding", "5px 10px")
			.style("border-radius", "5px")
			.style("font-size", "12px")
			.style("pointer-events", "none")
			.style("visibility", "hidden");

		// Dibujar círculos concéntricos
		for (let i = 1; i <= 5; i++) {
			svg.append("circle")
				.attr("r", (radius / 5) * i)
				.style("fill", "none")
				.style("stroke", "#ccc")
				.style("opacity", 0.5);
		}

		// Dibujar líneas radiales y etiquetas
		keys.forEach((key, i) => {
			const angle = angleSlice * i - Math.PI / 2; // Ángulo de cada eje
			svg.append("line")
				.attr("x1", 0)
				.attr("y1", 0)
				.attr("x2", radialScale(1) * Math.cos(angle))
				.attr("y2", radialScale(1) * Math.sin(angle))
				.style("stroke", "#ccc")
				.style("opacity", 0.5);

			svg.append("text")
				.attr("x", radialScale(1.2) * Math.cos(angle))
				.attr("y", radialScale(1.1, 5) * Math.sin(angle))
				.style("text-anchor", "middle")
				.style("font-size", "20px")
				.style("font-family", "Bebas Neue")
				.style("fill", "white")
				.text(key);
		});

		// Dibujar la red
		const points = keys.map((_, i) => {
			const angle = angleSlice * i - Math.PI / 2; // Ángulo de cada eje
			const x = radialScale(values[i]) * Math.cos(angle); // Coordenada X
			const y = radialScale(values[i]) * Math.sin(angle); // Coordenada Y
			return [x, y];
		});

		svg.append("polygon")
			.attr("points", points.map((d) => d.join(",")).join(" "))
			.style("fill", "rgba(128, 0, 128, 0.2)")
			.style("stroke", "#800080")
			.style("stroke-width", 2);

		// Dibujar puntos interactivos
		points.forEach(([x, y], i) => {
			svg.append("circle")
				.attr("cx", x)
				.attr("cy", y)
				.attr("r", 5)
				.style("fill", "#800080")
				.style("cursor", "pointer")
				.on("mouseover", () => {
					tooltip
						.style("visibility", "visible")
						.text(`${keys[i]}: ${values[i].toFixed(2)}`);
				})
				.on("mousemove", (event) => {
					tooltip
						.style("top", `${event.pageY - 10}px`)
						.style("left", `${event.pageX + 10}px`);
				})
				.on("mouseout", () => {
					tooltip.style("visibility", "hidden");
				});
		});
	}
	window.handleDiscoClick = function (event) {
		const songId = event.currentTarget.getAttribute("data-song-id");

		if (songId) {
			// Obtener los datos de la canción seleccionada
			currentSongData = songData[songId];

			// Redibujar el gráfico de radar con los datos de la canción seleccionada
			createRadarChart(currentSongData);

			// Actualizar la información de la canción actual (si el contenedor existe)
			const currentSongElement = document.getElementById("current-song");
			if (currentSongElement) {
				currentSongElement.innerHTML = `<p>${songId} - ${songData[songId].Artista}</p>`;
			}
		}
	};
</script>

<!-- Estructura contenido HTML -->
<main>
	<h1>Entre discos y datos: <br /> sintonías que nos conectan</h1>
	<h4>
		Descubrí cómo nuestras canciones favoritas se entrelazan, comparando
		nuestros ritmos, <br /> géneros y momentos musicales más escuchados del año.
	</h4>
	<div class="botones-filtro">
		<button
			on:click={() => (filtro = "Abril")}
			class:active={filtro === "Abril"}
		>
			Abril
		</button>
		<button
			on:click={() => (filtro = "Martin")}
			class:active={filtro === "Martin"}
		>
			Martín
		</button>
	</div>

	<div class="scroll-wrapper">
		<div class="image-container">
			<div class="scroll-container">
				<div
					class="disco-container"
					data-song-id="La Bachata"
					onclick="handleDiscoClick(event)"
				>
					<img
						src="public/images/Abril/bachata cerrado.png"
						alt="Disco cerrado"
						class="disco"
					/>
					<img
						src="public/images/Abril/bachata abierto.png"
						alt="Disco abierto"
						class="disco-hover"
					/>
				</div>
				<div
					class="disco-container"
					data-song-id="Milagrosa"
					onclick="handleDiscoClick(event)"
				>
					<img
						src="public/images/Abril/milagrosa cerrado.png"
						alt="Disco cerrado"
						class="disco"
					/>
					<img
						src="public/images/Abril/milagrosa abierto.png"
						alt="Disco abierto"
						class="disco-hover"
					/>
				</div>
				<div
					class="disco-container"
					data-song-id="Extasis"
					onclick="handleDiscoClick(event)"
				>
					<img
						src="public/images/Abril/extasis cerrado.png"
						alt="Disco cerrado"
						class="disco"
					/>
					<img
						src="public/images/Abril/extasis abierto.png"
						alt="Disco abierto"
						class="disco-hover"
					/>
				</div>
				<div
					class="disco-container"
					data-song-id="Rara vez"
					onclick="handleDiscoClick(event)"
				>
					<img
						src="public/images/Abril/rara vez cerrado.png"
						alt="Disco cerrado"
						class="disco"
					/>
					<img
						src="public/images/Abril/rara vez abierto.png"
						alt="Disco abierto"
						class="disco-hover"
					/>
				</div>
				<div
					class="disco-container"
					data-song-id="Baila Bach"
					onclick="handleDiscoClick(event)"
				>
					<img
						src="public/images/Abril/baila bach cerrado.png"
						alt="Disco cerrado"
						class="disco"
					/>
					<img
						src="public/images/Abril/baila bach abierto.png"
						alt="Disco abierto"
						class="disco-hover"
					/>
				</div>
				<div class="scroll-container">
					<div
						class="disco-container"
						data-song-id="Bien o Mal"
						onclick="handleDiscoClick(event)"
					>
						<img
							src="public/images/Martin/Bien o mal cerrado.png"
							alt="Disco cerrado"
							class="disco"
						/>
						<img
							src="public/images/Martin/Bien o mal abierto.png"
							alt="Disco abierto"
							class="disco-hover"
						/>
					</div>
					<div
						class="disco-container"
						data-song-id="Hood"
						onclick="handleDiscoClick(event)"
					>
						<img
							src="public/images/Martin/Hood cerrado.png"
							alt="Disco cerrado"
							class="disco"
						/>
						<img
							src="public/images/Martin/Hood abierto.png"
							alt="Disco abierto"
							class="disco-hover"
						/>
					</div>
					<div
						class="disco-container"
						data-song-id="Inmortal"
						onclick="handleDiscoClick(event)"
					>
						<img
							src="public/images/Martin/inmortal cerrado.png"
							alt="Disco cerrado"
							class="disco"
						/>
						<img
							src="public/images/Martin/inmortal abierto.png"
							alt="Disco abierto"
							class="disco-hover"
						/>
					</div>
					<div
						class="disco-container"
						data-song-id="Lo Tengo"
						onclick="handleDiscoClick(event)"
					>
						<img
							src="public/images/Martin/Lo tengo cerrado.png"
							alt="Disco cerrado"
							class="disco"
						/>
						<img
							src="public/images/Martin/Lo tengo abierto.png"
							alt="Disco abierto"
							class="disco-hover"
						/>
					</div>
					<div
						class="disco-container"
						data-song-id="M.A"
						onclick="handleDiscoClick(event)"
					>
						<img
							src="public/images/Martin/MA cerrado.png"
							alt="Disco cerrado"
							class="disco"
						/>
						<img
							src="public/images/Martin/MA abierto.png"
							alt="Disco abierto"
							class="disco-hover"
						/>
					</div>
				</div>
			</div>
		</div>

		<div class="image-container">
			<div class="scroll-container">
				<div
					class="disco-container"
					data-song-id="Bien o Mal"
					onclick="handleDiscoClick(event)"
				>
					<img
						src="public/images/Martin/Bien o mal cerrado.png"
						alt="Disco cerrado"
						class="disco"
					/>
					<img
						src="public/images/Martin/Bien o mal abierto.png"
						alt="Disco abierto"
						class="disco-hover"
					/>
				</div>
				<div
					class="disco-container"
					data-song-id="Inmortal"
					onclick="handleDiscoClick(event)"
				>
					<img
						src="public/images/Martin/inmortal cerrado.png"
						alt="Disco cerrado"
						class="disco"
					/>
					<img
						src="public/images/Martin/inmortal abierto.png"
						alt="Disco abierto"
						class="disco-hover"
					/>
				</div>
				<div
					class="disco-container"
					data-song-id="Hood"
					onclick="handleDiscoClick(event)"
				>
					<img
						src="public/images/Martin/Hood cerrado.png"
						alt="Disco cerrado"
						class="disco"
					/>
					<img
						src="public/images/Martin/Hood abierto.png"
						alt="Disco abierto"
						class="disco-hover"
					/>
				</div>
				<div
					class="disco-container"
					data-song-id="Lo Tengo"
					onclick="handleDiscoClick(event)"
				>
					<img
						src="public/images/Martin/Lo tengo cerrado.png"
						alt="Disco cerrado"
						class="disco"
					/>
					<img
						src="public/images/Martin/Lo tengo abierto.png"
						alt="Disco abierto"
						class="disco-hover"
					/>
				</div>
				<div
					class="disco-container"
					data-song-id="M.A"
					onclick="handleDiscoClick(event)"
				>
					<img
						src="public/images/Martin/MA cerrado.png"
						alt="Disco cerrado"
						class="disco"
					/>
					<img
						src="public/images/Martin/MA abierto.png"
						alt="Disco abierto"
						class="disco-hover"
					/>
				</div>
			</div>
		</div>
	</div>

	<br />
	<br />
	<br />
	<br />
	<br />
	<br />
	<svg id="radar-chart"></svg>
	<br />
	<br />
	<br />
	<br />
	<br />
	<br />
	<br />
	<br />
	<br />
	<br />

	<img class="gif2" src="public/images/tira.gif" alt="" />
	<h2>Nuestro perfil musical: Abril vs. Martín</h2>
	<p>
		Explorá nuestros géneros favoritos, artistas más escuchados y descubrí
		cómo se diferencian
		<br /> nuestros gustos. Una mirada general a lo que nos hace únicos a través
		de la música.
	</p>

	<div
		class="flourish-embed flourish-pictogram"
		data-src="visualisation/20491512"
	>
		<script
			src="https://public.flourish.studio/resources/embed.js"
		></script><noscript
			><img
				src="https://public.flourish.studio/visualisation/20491512/thumbnail"
				width="100%"
				alt="pictogram visualization"
			/></noscript
		>
	</div>
</main>

<!-- Estilos CSS -->

<style>
	.flourish-embed {
		background: transparent;
	}

	.scroll-wrapper {
		display: flex;
		gap: 50px;
		width: 100%;
		justify-content: center;
	}

	main {
		margin: 60px;
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		background: black;
		min-height: 100vh;
		color: white;
	}

	.botones-filtro {
		display: flex; /* Asegura que los botones estén alineados horizontalmente */
		justify-content: center;
		gap: 30px;
		margin-top: 20px;
		width: 100%;
		margin-bottom: 20px;
	}

	.botones-filtro button {
		font-family: Remora Sans W5;
		font-weight: 250;
		font-size: 15px;
		background-color: transparent;
		color: white;
		border: 2px solid #9747ff;
		border-radius: 20px;
		cursor: pointer;
		padding: 10px 20px;
		transition: all 0.3s ease;
	}

	.botones-filtro button:hover {
		background-color: #9747ff;
		color: white;
	}

	.botones-filtro button.active {
		background-color: #9747ff;
		color: white;
		border: 2px solid #9747ff;
	}

	h1 {
		font-family: Remora Sans W5;
		margin: 0;
		text-align: center;
		font-size: 45px;
	}

	h4 {
		font-family: Helvetica Neue LT Std;
		font-weight: lighter;
		text-align: center;
		font-size: 20px;
	}

	p {
		font-family: Helvetica Neue LT Std;
		font-weight: lighter;
		text-align: center;
		font-size: 17px;
	}

	button {
		font-family: Remora Sans W5;
		font-weight: 250;
		font-size: 15px;
		background-color: #9747ff;
		color: white;
		border-radius: 20px;
		cursor: pointer;
		padding: 10px 20px;
	}

	.mi-boton {
		font-family: Remora Sans W5;
		font-weight: 250;
		font-size: 15px;
		background-color: #9747ff;
		color: white;
		border-radius: 20px;
		cursor: pointer;
		padding: 10px 20px;
		transition: background-color 0.3s ease;
	}

	.mi-boton:hover {
		background-color: #6a2fb8;
	}

	#radar-chart {
		background: black;
		margin-top: 100px;
		overflow: visible;
	}

	.tooltip {
		visibility: hidden;
		position: absolute;
		z-index: 10;
		background: rgba(0, 0, 0, 0.7);
		color: white;
		padding: 5px 10px;
		border-radius: 5px;
		font-size: 12px;
		pointer-events: none;
	}

	.image-container {
		display: flex;
		gap: 200px;
		margin-top: 20px;
		justify-content: center;
	}

	.scroll-container {
		width: 350px;
		height: 350px;
		overflow-y: visible;
		overflow-x: hidden;
		margin: 0 auto;
		position: relative;
		padding: 20px;
	}

	.scroll-container::-webkit-scrollbar {
		display: none;
	}

	.disco-container {
		position: relative;
		z-index: 1;
		transition:
			transform 0.2s ease,
			z-index 0.2s ease;
	}

	.disco-container img {
		width: 100%;
		cursor: pointer;
		transition: transform 0.3s ease;
	}

	.disco-container:hover {
		transform: scale(1.1);
		box-shadow: 0px 0px 15px #9747ff;
		z-index: 10;
	}

	.disco {
		width: 193px;
		height: 200px;
		object-fit: contain;
		transition:
			transform 0.3s ease,
			box-shadow 0.3s ease;
	}

	.disco-hover {
		display: none;
	}

	.disco-container:hover .disco {
		display: none;
	}

	.disco-container:hover .disco-hover {
		display: block;
	}
</style>
