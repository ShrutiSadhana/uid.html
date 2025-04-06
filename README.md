<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Biological Data Parser</title>
  <style>
    body {
      font-family: Arial, Helvetica, sans-serif;
      background-color: #333;
      color: #fff;
      margin: 0;
    }

    header {
      background-color: #129423;
      padding: 10px 20px;
      display: flex;
      justify-content: space-between;
      align-items: start;
      width: 100%;
    }

    header h1 {
      margin: 0;
      font-size: 30px;
    }

    .menu-bar {
      display: flexbox;
      justify-content: space-between;
      align-items: flex-end;
      width: 100%;
    }

    .menu-bar button, .menu-bar select, .menu-bar a {
      padding: 10px 15px;
      background-color: #fff;
      color: #333;
      border: none;
      border-radius: 5px;
      cursor: pointer;
      margin: 5px;
    }

    .menu-bar button:hover, .menu-bar select:hover, .menu-bar a:hover {
      background-color: #129423;
    }

    .menu-bar a {
      text-decoration: none;
      color: #333;
    }

    .container {
      padding: 20px;
    }

    textarea {
      width: 100%;
      height: 200px;
      padding: 10px;
      font-size: 18px;
      border: 2px solid #fff;
      border-radius: 5px;
    }

    #output, #histogramOutput, #explanation {
      margin-top: 30px;
    }

    .histogram {
      margin-top: 20px;
      display: flex;
      align-items: center;
      flex-direction: column;
    }
    .histogram div {
      text-align: center;
      margin: 5px 0;
    }
    

    .histogram-bar {
      display: inline-block;
      height: 20px;
      border-radius: 5px;
      margin: 3px;
    }

    .gc-content-bar {
      background-color: #42b91a;
    }

    .sequence-length-bar {
      background-color: #be1c07;
    }

    .aa-content-bar {
      background-color: #ecad0c;
    }
    .x-axis {
      margin-top: 10px;
      display: flex;
      justify-content: space-between;
      width: 100%;
    }
    .x-axis span {
      text-align: center;
      font-size: 12px;
    }
    .y-axis {
      display: flex;
      justify-content: space-between;
      width: 100%;
    }
    .y-axis span {
      font-size: 12px;
      text-align: center;
    }

    .explanation {
      font-size: 14px;
      margin-top: 20px;
      color: #fff;
    }

    .highlighted-gc {
      color: rgb(223, 70, 9);
    }

    .highlighted-at {
      color: rgb(67, 125, 10);
    }

    .highlighted-aa {
      color: rgb(181, 158, 13);
    }
  </style>
</head>
<body>

  <header>
    <div class="menu-bar">
      <h1>Biological Data Parser</h1>
      <div>
        <button onclick="parseSequence()">Parse Sequence</button>
        <button onclick="toggleHistogram()">Toggle Histogram</button>
        <a href="https://www.ncbi.nlm.nih.gov" target="_blank">Visit NCBI</a>
        <select id="saveOptions" onchange="saveOutput()">
          <option value="">Save Output</option>
          <option value="csv">CSV</option>
          <option value="json">JSON</option>
        </select>
        <button onclick="clearAll()">Clear All</button>
        <button onclick="showHelp()">Help</button>
      </div>
    </div>
  </header>

  <div class="container">
    <label for="sequenceInput">Enter your sequence (manual input):</label>
    <textarea id="sequenceInput" placeholder="Type or paste your sequence here..."></textarea>

    <br><br>

    <label for="fileInput">Or upload your sequence (FASTA/GenBank):</label>
    <input type="file" id="fileInput" accept=".fasta,.gb,.gbk" onchange="handleFileUpload(event)">

    <br><br>

    <button onclick="parseSequence()">Parse Sequence</button>
  </div>

  <div id="output">
    <!-- Parsed result will be displayed here -->
  </div>

  <div id="histogramOutput">
    <!-- Histogram will be displayed here -->
  </div>

  <div id="explanation" class="explanation">
    <!-- Explanation for understanding the plot -->
  </div>

  <script>
    let parsedData = {};

    function handleFileUpload(event) {
      const file = event.target.files[0];
      if (!file) return;

      const reader = new FileReader();
      reader.onload = function(e) {
        const content = e.target.result;
        parseSequenceFromFile(content, file.name);
      };
      reader.readAsText(file);
    }

    function parseSequenceFromFile(content, fileName) {
      let sequence = '';
      let annotations = '';

      if (fileName.endsWith('.fasta')) {
        // Parse FASTA file: Extract sequence and header
        const lines = content.split('\n');
        sequence = lines.filter(line => !line.startsWith('>')).join('').replace(/\s/g, '');
        annotations = lines.filter(line => line.startsWith('>')).join('\n');
      } else if (fileName.endsWith('.gb') || fileName.endsWith('.gbk')) {
        // Parse GenBank file
        const featureRegex = /\/gene="([^"]+)"/g;
        const matches = [...content.matchAll(featureRegex)];
        annotations = matches.map(match => match[1]).join(', ') || 'No annotations found';
        const sequenceMatch = content.match(/ORIGIN([\s\S]+?)\/\//);
        if (sequenceMatch) {
          sequence = sequenceMatch[1].replace(/\s+/g, ''); // clean up sequence
        }
      }

      parsedData = { sequence, annotations };
      displayParsedData(sequence, annotations);
       generateHistogram(sequence);
    }

    function parseSequence() {
      const sequence = document.getElementById('sequenceInput').value.trim();
      if (!sequence) {
        alert("Please enter or upload a sequence.");
        return;
      }

      const annotations = "No annotations available for manual input";
      parsedData = { sequence, annotations };
      displayParsedData(sequence, annotations);
      generateHistogram(sequence);
    }

    function displayParsedData(sequence, annotations) {
      const highlightedSequence = highlightSequence(sequence);
      const parsedResult = `
        <h2>Parsed Data:</h2>
        <p><strong>Sequence:</strong> ${highlightedSequence}</p>
        <p><strong>Annotations:</strong> ${annotations}</p>
      `;
      document.getElementById('output').innerHTML = parsedResult;
    }

    function highlightSequence(sequence) {
      const uniqueAAs = new Set(sequence.split('').filter(base => 'ACDEFGHIKLMNPQRSTVWXY'.includes(base.toUpperCase())));
      return sequence.split('').map(base => {
        if (base.toUpperCase() === 'G' || base.toUpperCase() === 'C') {
          return `<span class="highlighted-gc">${base}</span>`;
        } else if (base.toUpperCase() === 'A' || base.toUpperCase() === 'T') {
          return `<span class="highlighted-at">${base}</span>`;
        } else if (uniqueAAs.has(base.toUpperCase())) {
          return `<span class="highlighted-aa">${base}</span>`;
        }
        return base; // Return other characters without styling
      }).join('');
    }

    function generateHistogram(sequence) {
      const gcCount = (sequence.match(/[GCgc]/g) || []).length; // Count G or C
      const gcContent = (gcCount / sequence.length) * 100;
      const sequenceLength = sequence.length;

      const aminoAcidCount = countAminoAcids(sequence);  // Count amino acids
      const aaHistogramData = getAminoAcidHistogramData(aminoAcidCount, sequenceLength);

      const histogram = document.getElementById('histogramOutput');
      let histogramHTML = `<h3>Sequence Statistics</h3><div class="histogram">`;

      histogramHTML += `
        <div>
          <div class="histogram-bar gc-content-bar" style="width: ${gcContent}%"></div>
          <p>GC Content: ${gcContent.toFixed(2)}%</p>
        </div>
        <div>
          <div class="histogram-bar sequence-length-bar" style="width: ${sequenceLength / 10}px"></div>
          <p>Length: ${sequenceLength}</p>
        </div>
      `;

      histogramHTML += `<h3>Amino Acid Distribution</h3>`;
      Object.keys(aaHistogramData).forEach(aa => {
        if (aaHistogramData[aa].count > 0) {
          histogramHTML += `
            <div>
              <div class="histogram-bar aa-content-bar" style="width: ${aaHistogramData[aa].percentage}%;"></div>
              <p>${aa}: ${aaHistogramData[aa].count} (${aaHistogramData[aa].percentage.toFixed(2)}%)</p>
            </div>
          `;
        }
      });

      histogramHTML += `</div>`;
      histogramHTML += `
        <div class="x-axis">
          <span>0%</span>
          <span>100%</span>
        </div>
      `;

      histogramHTML += `
        <div class="y-axis">
          <span>GC Content</span>
          <span>Sequence Length</span>
        </div>
      `;

      histogram.innerHTML = histogramHTML;

      const explanationText = `
        <p><strong>How to interpret the plot:</strong></p>
        <ul>
          <li><strong>GC Content Bar:</strong> This bar shows the percentage of guanine (G) and cytosine (C) bases in the sequence.</li>
          <li><strong>Sequence Length Bar:</strong> This bar shows the total length of the sequence.</li>
          <li><strong>Amino Acid Distribution:</strong> The distribution of amino acids and their respective percentages in the sequence (only showing those present in the sequence).</li>
        </ul>
      `;
      document.getElementById('explanation').innerHTML = explanationText;
    }

    // Function to count amino acids in the sequence
    function countAminoAcids(sequence) {
      const aminoAcids = 'ACDEFGIKLMNPQRSTVWXY'; // Standard amino acids
      const counts = {};

      // Initialize counts for all amino acids
      aminoAcids.split('').forEach(aa => {
        counts[aa] = 0;
      });

      // Count each amino acid in the sequence
      sequence.split('').forEach(base => {
        if (aminoAcids.includes(base)) {
          counts[base]++;
        }
      });

      return counts;
    }

    // Function to get amino acid histogram data (percentage and count)
    function getAminoAcidHistogramData(aminoAcidCount, sequenceLength) {
      const aaData = {};
      for (let aa in aminoAcidCount) {
        aaData[aa] = {
          count: aminoAcidCount[aa],
          percentage: (aminoAcidCount[aa] / sequenceLength) * 100
        };
      }
      return aaData;
    }

    function toggleHistogram() {
      const histogram = document.getElementById('histogramOutput');
      histogram.style.display = (histogram.style.display === 'none' || histogram.style.display === '') ? 'block' : 'none';
    }

    function clearAll() {
      document.getElementById('sequenceInput').value = '';
      document.getElementById('fileInput').value = '';
      document.getElementById('output').innerHTML = '';
      document.getElementById('histogramOutput').innerHTML = '';
      document.getElementById('explanation').innerHTML = '';
    }

    function showHelp() {
      alert('Help: Enter a biological sequence to parse and analyze. You can upload files in FASTA or GenBank formats.');
    }

    function saveOutput() {
      const format = document.getElementById('saveOptions').value;
      let content = '';

      if (format === 'csv') {
        content = 'Sequence, Annotations\n';
        content += `${parsedData.sequence}, ${parsedData.annotations}`;
      } else if (format === 'json') {
        content = JSON.stringify(parsedData, null, 2);
      }

      const blob = new Blob([content], { type: 'text/plain' });
      const url = URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      a.download = `sequence.${format}`;
      document.body.appendChild(a);
      a.click();
      document.body.removeChild(a);
    }
  </script>
</body>
</html>
