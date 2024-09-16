<script>
	import { onMount } from 'svelte';
	let editor;
	let code = localStorage.getItem('umlCode') ||
	  `class Person {
		+String name
		+int age
		+String address
		+getName(): String
		+getAge(): int
		+setName(name: String): void
		+setAge(age: int): void
	  }
	  class Employee {
		+String jobTitle
		+double salary
		+getJobTitle(): String
		+getSalary(): double
	  }
	  Person <|-- Employee : Inherits`;
  
	let classes = JSON.parse(localStorage.getItem('classPositions')) || [];
	let relationships = JSON.parse(localStorage.getItem('relationships')) || [];
	let scale = 1;
	let showHelp = false;
  
	onMount(() => {
	  const script = document.createElement('script');
	  script.src = 'https://cdn.jsdelivr.net/npm/monaco-editor@0.33.0/min/vs/loader.js';
	  script.onload = () => {
		window.require.config({ paths: { vs: 'https://cdn.jsdelivr.net/npm/monaco-editor@0.33.0/min/vs' } });
		window.require(['vs/editor/editor.main'], () => {
		  editor = window.monaco.editor.create(document.getElementById('editor'), {
			value: code,
			language: 'plaintext',
			theme: 'vs-light',
			automaticLayout: true,
		  });
  
		  // Listen to changes in the editor
		  editor.onDidChangeModelContent(() => {
			const currentCode = editor.getValue();
			localStorage.setItem('umlCode', currentCode);
			generateDiagram(); // Dynamically update diagram when content changes
		  });
  
		  // Initial diagram generation
		  generateDiagram();
		});
	  };
	  document.head.appendChild(script);
	});
  
	function parseClassDiagram(text) {
	  const classRegex = /class (\w+)\s*{([^}]*)}/g;
	  const relationshipRegex = /(\w+)\s*<\|\--\s*(\w+)\s*:\s*(\w+)/g;
  
	  const parsedClasses = [];
	  let match;
  
	  // Parse classes and their attributes/methods
	  while ((match = classRegex.exec(text)) !== null) {
		const className = match[1];
		const body = match[2].trim().split("\n");
  
		const attributes = [];
		const methods = [];
  
		for (const line of body) {
		  const cleaned = line.trim();
		  if (cleaned.includes("()")) {
			methods.push(cleaned);
		  } else {
			attributes.push(cleaned);
		  }
		}
  
		// Check if class position is already saved
		const savedClass = classes.find(c => c.name === className);
		const x = savedClass ? savedClass.x : 50 + parsedClasses.length * 200;
		const y = savedClass ? savedClass.y : 50;
  
		// Add class to the parsedClasses array
		parsedClasses.push({ name: className, attributes, methods, x, y });
	  }
  
	  const parsedRelationships = [];
  
	  // Parse relationships between classes
	  while ((match = relationshipRegex.exec(text)) !== null) {
		parsedRelationships.push({ from: match[2], to: match[1], type: match[3] });
	  }
  
	  return { classes: parsedClasses, relationships: parsedRelationships };
	}
  
	function generateDiagram() {
	  try {
		const userCode = editor ? editor.getValue() : code;
		const { classes: parsedClasses, relationships: parsedRelationships } = parseClassDiagram(userCode);
  
		classes = parsedClasses;
		relationships = parsedRelationships;
  
		// Save class positions and relationships to localStorage
		localStorage.setItem('classPositions', JSON.stringify(classes));
		localStorage.setItem('relationships', JSON.stringify(relationships));
	  } catch (err) {
		alert('Error parsing input. Please check the format.');
	  }
	}
  
	let dragging = null;
  
	function startDrag(index, event) {
	  if (index < 0 || index >= classes.length) {
		return; // Index out of bounds
	  }
	  dragging = { index, offsetX: event.clientX - classes[index].x, offsetY: event.clientY - classes[index].y };
	}
  
	function stopDrag() {
	  dragging = null;
  
	  // Save the updated class positions to localStorage
	  localStorage.setItem('classPositions', JSON.stringify(classes));
	}
  
	function drag(event) {
	  if (dragging !== null) {
		const { index, offsetX, offsetY } = dragging;
		if (index < 0 || index >= classes.length) {
		  return; // Index out of bounds
		}
		classes[index].x = event.clientX - offsetX;
		classes[index].y = event.clientY - offsetY;
	  }
	}
  
	function getEdgeCoordinates(fromClass, toClass) {
	  const fromX = fromClass.x + 200;
	  const fromY = fromClass.y + 50;
	  const toX = toClass.x;
	  const toY = toClass.y + 50;
  
	  const controlPointX1 = fromX + (toX - fromX) / 2;
	  const controlPointY1 = fromY - 50;
	  const controlPointX2 = controlPointX1;
	  const controlPointY2 = toY - 50;
  
	  return {
		fromX,
		fromY,
		toX,
		toY,
		controlPointX1,
		controlPointY1,
		controlPointX2,
		controlPointY2,
	  };
	}
  
	// Zoom functions
	function zoomIn() {
	  scale += 0.1;
	}
  
	function zoomOut() {
	  scale = Math.max(0.1, scale - 0.1);
	}
  
	function resetZoom() {
	  scale = 1;
	}
  
	// Handle mouse wheel zoom
	function handleWheel(event) {
	  event.preventDefault(); // Prevent default scrolling behavior
	  if (event.deltaY < 0) {
		zoomIn();
	  } else {
		zoomOut();
	  }
	}
  
	function toggleHelp() {
	  showHelp = !showHelp;
	}
  </script>  <!-- Template and styling sections remain the same as your code -->


  
  <div class="header">
	<h1 >UML Diagrams Generator</h1>
	<button class="help-btn" on:click={toggleHelp}>Help</button>
  </div>
  
  {#if showHelp}
	<div class="help-dialog">
	  <div class="help-content">
		<h2>Help Instructions</h2>
		<p>1. Input Code: Write your UML class diagram in the input area on the left. Use the following format:</p>
		<pre>
  class ClassName 
	+attributeName: type
	+methodName(): returnType
  
  class AnotherClass 
	+anotherAttribute: type
	+anotherMethod(): returnType
  
  ClassName |-- AnotherClass : Inherits
		</pre>
		<p>2. Generate Diagram: Click the "Generate Diagram" button to create the UML diagram based on your input.</p>
		<p>3. Drag and Drop: You can drag and reposition the class boxes in the diagram.</p>
		<p>4. Zoom: Use the zoom buttons to zoom in, zoom out, or reset the zoom level.</p>
		<button class="close-btn" on:click={toggleHelp}>Close</button>
	  </div>
	</div>
  {/if}
  
  <div class="container">
	<div class="left">
	  <div class="editor-container">
		<h2>Input Code</h2>
		<div id="editor" style="height:85%; border: 1px solid #ccc;"></div>
		<button class="generate-btn" on:click={generateDiagram}>Generate Diagram</button>
	  </div>
	</div>
  
	<div class="uml-canvas" on:mousemove={drag} on:mouseup={stopDrag} on:wheel={handleWheel} style="transform: scale({scale}); transform-origin: top left;">
	  {#each classes as Class, i}
		<div 
		
		  class="class-box" 
		  style="top: {Class.y}px; left: {Class.x}px;" 
		  on:mousedown={(event) => startDrag(i, event)}
		>
		  <div class="class-name">{Class.name}</div>
		  <div class="class-attributes">
			{#each Class.attributes as attribute}
			  <div>{attribute}</div>
			{/each}
		  </div>
		  <div class="class-methods">
			{#each Class.methods as method}
			  <div>{method}</div>
			{/each}
		  </div>
		</div>
	  {/each}
  
	  <svg style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;">
		<defs>
		  <marker id="arrowhead" markerWidth="10" markerHeight="7" refX="10" refY="3.5" orient="auto">
			<polygon points="0 0, 10 3.5, 0 7" fill="black" />
		  </marker>
		</defs>
  
		{#each relationships as relationship}
		  {#if classes.some(classItem => classItem.name === relationship.from) && classes.some(classItem => classItem.name === relationship.to)}
			{#each classes as fromClass (fromClass.name)}
			  {#if fromClass.name === relationship.from}
				{#each classes as toClass (toClass.name)}
				  {#if toClass.name === relationship.to}
					<path
					  d="M{getEdgeCoordinates(fromClass, toClass).fromX} {getEdgeCoordinates(fromClass, toClass).fromY}
						C{getEdgeCoordinates(fromClass, toClass).controlPointX1} {getEdgeCoordinates(fromClass, toClass).controlPointY1},
						 {getEdgeCoordinates(fromClass, toClass).controlPointX2} {getEdgeCoordinates(fromClass, toClass).controlPointY2},
						 {getEdgeCoordinates(fromClass, toClass).toX} {getEdgeCoordinates(fromClass, toClass).toY}"
					  stroke="black"
					  stroke-width="2"
					  fill="transparent"
					  marker-end="url(#arrowhead)"
					/>
				  {/if}
				{/each}
			  {/if}
			{/each}
		  {/if}
		{/each}
	  </svg>
	</div>
  
	<!-- Styled Zoom Buttons -->
	<div class="zoom-controls">
	  <button class="zoom-btn" on:click={zoomIn}>
		<svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24">
		  <path d="M12 5v7h7v2h-7v7h-2v-7h-7v-2h7v-7h2z"/>
		</svg>
	  </button>
	  <button class="zoom-btn" on:click={zoomOut}>
		<svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24">
		  <path d="M19 13H5v-2h14v2z"/>
		</svg>
	  </button>
	  <button class="zoom-btn" on:click={resetZoom}>
		<svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24">
		  <path d="M12 4.354v2.382c3.128.257 5.646 2.475 6.715 5.552h-1.95c-1.229-2.2-3.44-3.693-6.465-3.83v1.954h2.918c-.253 1.694-1.02 3.184-2.209 4.228-1.478 1.31-3.52 2.046-5.703 2.038-1.283-.006-2.542-.313-3.677-.898l-2.305 2.31c1.481.695 3.101 1.111 4.809 1.24v-2.381c-2.2-.257-4.223-1.168-5.734-2.788-.601-.597-1.095-1.275-1.527-2.016l2.347-2.348c.546.645 1.177 1.22 1.869 1.709 1.34.845 2.868 1.391 4.438 1.452v-2.377c-1.212-.044-2.402-.446-3.471-1.189-1.517-1.03-2.508-2.464-2.995-4.016h1.972c.479 1.226 1.058 2.405 1.818 3.457.978 1.279 2.151 2.361 3.553 3.17z"/>
		</svg>
	  </button>
	</div>
  </div>
  
  <style>
	.header {
		position: fixed; /* Fixes the header at the top */
		top: 0;
		left: 0;
		width: 100%; /* Ensures the header spans the full width */
		background-color: #ff6f3f; /* Light blue color */
		padding: 10px;
		display: flex;
		justify-content: space-between;
		align-items: center;
		box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
		z-index: 1000; /* Ensures the header stays above other elements */
	}

	.header h1 {
		margin: 0;
		text-align: center;
		margin-left: 45%;
		font-size: 1.5em;
		color: #ffffff;
	}

	.help-btn {
		background-color: #ffffff;
		margin-right: 5%;
		color: rgb(0, 0, 0);
		border: none;
		padding: 10px 20px;
		cursor: pointer;
		border-radius: 5px;
		font-size: 0.9em;
	}

	.help-btn:hover {
		background-color: #98b5ec;
	}

	.help-dialog {
		position: fixed;
		top: 20%;
		left: 20%;
		width: 60%;
		background-color: white;
		border: 1px solid #ccc;
		box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
		z-index: 1000;
		padding: 20px;
		border-radius: 8px;
	}

	.help-content {
		max-height: 400px;
		overflow-y: auto;
	}

	.help-dialog h2 {
		margin-top: 0;
	}

	.close-btn {
		background-color: #ff6f3f; /* Red-orange color */
		color: white;
		border: none;
		padding: 10px 20px;
		cursor: pointer;
		border-radius: 5px;
		font-size: 0.9em;
		margin-top: 10px;
	}

	.close-btn:hover {
		background-color: #e64a19;
	}

	.class-box {
		position: absolute;
		width: 200px;
		border: 1px solid #000;
		background-color: white;
		border-radius: 5px;
		box-shadow: 2px 2px 5px rgba(0, 0, 0, 0.1);
		cursor: grab;
		transition: box-shadow 0.3s;
	}

	.class-box:hover {
		box-shadow: 4px 4px 8px rgba(0, 0, 0, 0.2);
	}

	.class-name {
		background-color: #333;
		color: white;
		padding: 5px;
		text-align: center;
		font-weight: bold;
	}

	.class-attributes, .class-methods {
		padding: 5px;
	}

	.class-attributes div, .class-methods div {
		padding: 2px 0;
	}

	svg {
		pointer-events: none;
	}

	.container {
		display: flex;
		margin-top: 70px; /* Adjusted to prevent overlap with the fixed header */
	}

	.left {
		position: fixed;
		top: 70px; /* Adjusted to fit below the fixed header */
		left: 0;
		width: 30%; 
		height: calc(100vh - 70px); /* Adjusted height to account for the header */
		background-color: #f9f9f9;
		padding: 10px;
		box-shadow: 2px 0 5px rgba(0, 0, 0, 0.1);
	}

	.editor-container {
		display: flex;
		flex-direction: column;
		height: 100%;
	}

	.editor-container h2 {
		margin: 0;
		font-size: 1.2em;
		color: #333;
	}

	.uml-canvas {
		margin-left: 30%; 
		width: 100%;
		height: calc(100vh - 70px); 
		background-color: #f0f0f0;
		position: relative;
		overflow: auto;
	}



	.zoom-controls {
		position: fixed;
		bottom: 20px;
		right: 20px;
		display: flex;
		flex-direction: column;
		gap: 10px;
	}

	.zoom-btn {
		background-color: #ff6f3f; /* Red-orange color */
		color: white;
		width: 40px;
		height: 40px;
		border: none;
		border-radius: 50%;
		cursor: pointer;
		display: flex;
		justify-content: center;
		align-items: center;
		box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
		transition: background-color 0.3s, transform 0.2s;
	}

	.zoom-btn svg {
		width: 20px;
		height: 20px;
		fill: white;
	}

	.zoom-btn:hover {
		background-color: #43a4e5; /* Darker shade on hover */
		transform: scale(1.05); /* Slightly enlarge on hover */
	}

	.zoom-btn:active {
		transform: scale(0.95); /* Shrinks slightly when clicked */
	}

	.generate-btn {
		margin-top: 10px;
		background-color: #ff6f3f; /* Red-orange color */
		color: white;
		padding: 20px 25px;
		font-size: 14px;
		font-weight: bold;
		border: none;
		border-radius: 5px;
		cursor: pointer;
		transition: background-color 0.3s;
	}

	.generate-btn:hover {
		background-color: #4384e5; /* Darker shade on hover */
	}

	.generate-btn:active {
		transform: scale(0.95); /* Shrinks slightly when clicked */
	}
</style>