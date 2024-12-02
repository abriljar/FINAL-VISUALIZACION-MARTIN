<!-- Script JS -->
<script>
	import * as d3 from "d3";
	import { onMount } from "svelte";

	let width = 400;
	let height = 400;
	let filtro = "Martin";
	let filteredData = [];
	let originalData = [];
	let songData = {}; // Datos de las canciones mapeadas por nombre
	let currentSongData = {}; // Datos actuales para el gráfico

	const energyRotationSpeeds = {
		Alta: "2s", // Rápida
		Media: "6s", // Media
		Baja: "16s", // Lenta
	};
	$: {
		if (filteredData.length > 0) {
			// Create a filtered copy of the data
			filteredData = originalData.filter((p) => p.Usuario == filtro);
			console.log("Filtered Data:", filteredData);
		}
	}

	onMount(async () => {
		// Cargar datos desde el archivo CSV
		originalData = await d3.csv("/data/Spotify.csv");
		filteredData = originalData;
		console.log(filteredData);
		const groupedData = Array.from(
			d3.group(originalData, (d) => d.Usuario),
			([nombre, canciones]) => ({ nombre, canciones }),
		);
		console.log(groupedData);
		songData = originalData.reduce((acc, song) => {
			acc[song.Nombre] = {
				Acustica: +song.Acustica,
				Bailabilidad: +song.Bailabilidad,
				Energy: song.Energy.trim(), // Normalizar energía también
				Instrumentalidad: +song.Instrumentalidad,
				Vivacidad: +song.Vivacidad,
				Palabras: +song.Palabras,
				Artista: song.Artista,
				Usuario: song.Usuario,
				Valencia: +song.Valencia,
			};
			return acc;
		}, {});

		// Cargar los datos iniciales del gráfico
		if (originalData.length > 0) {
			currentSongData = Object.values(songData)[0]; // Primera canción como inicial
			createRadarChart(currentSongData);
		}

		console.log("song data", songData);
	});

	console.log("filter afuera del onmout", filteredData);

	function createRadarChart(data) {
		d3.select("#radar-chart").selectAll("*").remove(); // Limpiar el gráfico previo
		const relevantKeys = [
			"Valencia",
			"Instrumentalidad",
			"Vivacidad",
			"Acustica",
			"Bailabilidad",
			"Palabras",
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
			.style("fill", "rgba(249, 139, 11, 0.5)")
			.style("stroke", "#F98B0B")
			.style("stroke-width", 2);

		// Dibujar puntos interactivos
		points.forEach(([x, y], i) => {
			svg.append("circle")
				.attr("cx", x)
				.attr("cy", y)
				.attr("r", 5)
				.style("fill", "#F98B0B")
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
			const discoGrandeImg = document.getElementById("disco-grande");
			discoGrandeImg.src = `/images/Martin/Disco grande ${songId.toLowerCase().replace(/\s+/g, "-")}.png`;

			const songEnergy = songData[songId]?.Energy || "Media";
			const rotationSpeed = energyRotationSpeeds[songEnergy];

			discoGrandeImg.style.animationDuration = rotationSpeed;

			currentSongData = songData[songId];
			createRadarChart(currentSongData);
			document.getElementById("current-song").innerHTML =
				`<p>${songId} - ${songData[songId].Artista}</p>`;
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
		on:click={() => {
			window.location.href = "https://final-visualizacion-abril.vercel.app/";
		}}
	>
		Abril
	</button>
		<button
			on:click={() => {
				filtro = "Martin";
				console.log("filtro", filtro);
			}}
			class:active={filtro === "Martin"}
		>
			Martín
		</button>
	</div>

	<div class="image-container">
		<div class="scroll-gif">
			<div class="scroll-container">
				<div
					class="disco-container"
					data-song-id="Inmortal"
					onclick="handleDiscoClick(event)"
				>
					<img
						src="/images/Martin/inmortal cerrado.png"
						alt="Disco cerrado"
						class="disco"
					/>
					<img
						src="/images/Martin/inmortal abierto.png"
						alt="Disco abierto"
						class="disco-hover"
					/>
				</div>
				<div
					class="disco-container"
					data-song-id="Bien o Mal"
					onclick="handleDiscoClick(event)"
				>
					<img
						src="/images/Martin/Bien o mal cerrado.png"
						alt="Disco cerrado"
						class="disco"
					/>
					<img
						src="/images/Martin/Bien o mal abierto.png"
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
						src="/images/Martin/Hood cerrado.png"
						alt="Disco cerrado"
						class="disco"
					/>
					<img
						src="/images/Martin/Hood abierto.png"
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
						src="/images/Martin/MA cerrado.png"
						alt="Disco cerrado"
						class="disco"
					/>
					<img
						src="/images/Martin/MA abierto.png"
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
						src="/images/Martin/Lo tengo cerrado.png"
						alt="Disco cerrado"
						class="disco"
					/>
					<img
						src="/images/Martin/Lo tengo abierto.png"
						alt="Disco abierto"
						class="disco-hover"
					/>
				</div>
			</div>
			<img class="gif" src="/images/Comp 1_1.gif" alt="" />
		</div>

		<div class="containerVinilo">
			<img class="pua" src="/images/Group.png" alt="" />
			<div class="discoGrande">
				<img
					id="disco-grande"
					src="/images/Martin/Disco grande inmortal.png"
					alt="Disco grande vinilo"
				/>
			</div>

			<div class="discoynombre">
				<img
					src="/images/music.png"
					alt="Imagen 2"
					class="image2"
				/>
				<div id="current-song" class="current-song"></div>
			</div>
		</div>
	</div>

	<h2>Análisis de la canción</h2>
	<p>
		Acá vas a poder ver el análisis de las canciones, con los datos
		proporcionados por el Spotify Wrapped 2023.
	</p>

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
	<div class="call-to-action">
		<h2>
			¿Te animás a ver quién la rompe más <br /> con su selección musical?
		</h2>
		<p>
			Las 5 canciones de Abril se enfrentan a las 5 canciones de Martín.
			Hacé clic y descubrí quién tiene el mejor repertorio. <br /> ¡Las comparaciones
			están listas, solo faltás vos! ¡Dale play y sorprendete!
		</p>
		<a href="https://comparativaspotify.vercel.app/" class="mi-boton">Ver comparativa de canciones</a>
	</div>
</main>

<!-- Estilos CSS -->

<style>
	.pua {
		width: 35%;
		height: 60%;
		position: absolute;
		top: 38%;
		left: 74%;
		transform: translate(-50%, -50%);
		z-index: 3;
	}

	.scroll-gif {
		justify-content: center;
		align-items: center;
		display: flex;
		flex-direction: column;
		width: 40%;
		height: 40%;
	}

	.gif {
		height: 30%;
		width: 65%;
		margin-top: 20px;
	}

	.discoGrande img {
		animation: spin linear infinite;
	}

	@keyframes spin {
		from {
			transform: rotate(0deg);
		}
		to {
			transform: rotate(360deg);
		}
	}

	.containerVinilo {
		position: relative;
		width: 600px;
		height: auto;
	}

	.discoGrande {
		width: 77%;
		height: 77%;
		position: absolute;
		top: 50.73%;
		left: 50%;
		transform: translate(-50%, -50%);
		z-index: 2;
	}

	.discoynombre {
		display: flex;
		flex-direction: column;
		align-items: center;
		gap: 10px;
		margin-top: 20px;
	}

	.current-song {
		margin-top: 30px;
		text-align: center;
		font-size: 30px;
		color: white;
		font-family: Bebas Neue;
		font-weight: bold;
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

	h2 {
		font-family: Remora Sans W5;
		margin-top: 80px;
		text-align: center;
		font-size: 30px;
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
		text-decoration: none;
	}

	.mi-boton:hover {
		background-color: #6a2fb8;
	}

	.call-to-action {
		background-color: #2f2f2f;
		border-radius: 300px;
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		text-align: center;
		padding: 20px;
		gap: 20px;
		max-width: 1000px;
		color: white;
	}

	.call-to-action h2 {
		margin: 0;
		font-size: 24px;
		line-height: 1.5; /* Mejora la legibilidad */
	}

	.call-to-action p {
		margin: 0;
		font-size: 16px;
		line-height: 1.5;
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
		gap: 100px;
		justify-content: center;
		align-items: center;
	}

	.image2 {
		width: auto;
		height: 600px;
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
