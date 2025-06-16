<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <title>Simulador de Movimiento</title>
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/mathjs/11.9.0/math.min.js"></script>
  <style>
    * {
      box-sizing: border-box;
    }

    body {
      margin: 0;
      font-family: 'Segoe UI', sans-serif;
      background: linear-gradient(135deg, #e0f7fa, #fce4ec);
      color: #2f3640;
    }

    header {
      background: #2f3640;
      color: white;
      text-align: center;
      padding: 40px 20px;
      box-shadow: 0 4px 10px rgba(0, 0, 0, 0.1);
    }

    header h1 {
      margin: 0;
      font-size: 2rem;
      letter-spacing: 1px;
    }

    main {
      max-width: 960px;
      margin: 40px auto;
      background: #ffffff;
      padding: 40px;
      border-radius: 16px;
      box-shadow: 0 10px 25px rgba(0, 0, 0, 0.1);
    }

    label {
      font-weight: bold;
      display: block;
      margin: 20px 0 6px;
    }

    select, input[type="number"], input[type="text"] {
      width: 100%;
      max-width: 500px;
      padding: 12px;
      font-size: 1rem;
      border-radius: 8px;
      border: 1px solid #ccc;
      background: #f9f9f9;
      transition: all 0.2s ease-in-out;
    }

    select:focus, input:focus {
      outline: none;
      border-color: #44bd32;
      background: #fff;
    }

    button {
      margin-top: 25px;
      background: #44bd32;
      color: white;
      border: none;
      padding: 14px 28px;
      font-size: 16px;
      border-radius: 10px;
      cursor: pointer;
      transition: background 0.3s ease;
    }

    button:hover {
      background: #4cd137;
    }

    .simulador {
      position: relative;
      height: 80px;
      background: #dcdde1;
      border-radius: 12px;
      margin: 40px 0;
      overflow: hidden;
      border: 2px dashed #ccc;
    }

    .objeto {
      position: absolute;
      top: 50%;
      left: 0;
      transform: translateY(-50%);
      width: 36px;
      height: 36px;
      background: #e84118;
      border-radius: 50%;
      box-shadow: 0 6px 12px rgba(0, 0, 0, 0.25);
      transition: left 0.1s linear;
    }

    canvas {
      margin-top: 30px;
      background: #fff;
      padding: 12px;
      border-radius: 10px;
      box-shadow: 0 6px 16px rgba(0, 0, 0, 0.08);
    }

    small {
      color: #777;
      margin-top: 5px;
      display: block;
      font-size: 0.9em;
    }
  </style>
</head>
<body>
  <header>
    <h1>Simulador de Movimiento (MRU, MRUV, MRV, Personalizado)</h1>
  </header>

  <main>
    <label for="tipo">Tipo de Movimiento:</label>
    <select id="tipo">
      <option value="mru">MRU (velocidad constante)</option>
      <option value="mruv">MRUV (aceleración constante)</option>
      <option value="mrv">MRV (aceleración variable)</option>
      <option value="personalizado">Fórmula personalizada</option>
    </select>

    <div id="parametros"></div>

    <div id="formulapersonalizada" style="display:none;">
      <label>Fórmulas disponibles:</label>
      <select onchange="document.getElementById('formula').value=this.value">
        <option value="5*t^2 + 3*sin(t)">5*t² + 3·sin(t)</option>
        <option value="10*t + 5*sin(t)">10·t + 5·sin(t)</option>
        <option value="2*t^3 - t^2 + 4">2·t³ - t² + 4</option>
      </select>

      <label>Fórmula de posición en función del tiempo (t):</label>
      <input type="text" id="formula" value="5*t^2 + 3*sin(t)">
      <small>Usa notación como: <code>t^2</code>, <code>sin(t)</code>, etc.</small>
    </div>

    <button onclick="iniciarSimulacion()">Iniciar Simulación</button>

    <div class="simulador">
      <div class="objeto" id="objeto"></div>
    </div>

    <canvas id="graficaPos"></canvas>
    <canvas id="graficaVel"></canvas>
    <canvas id="graficaAcel"></canvas>
  </main>

  <script>
    let intervalo, tiempo = 0;
    let datosPos = [], datosVel = [], datosAcel = [];
    let graficaPos, graficaVel, graficaAcel;

    const tipoMovimiento = document.getElementById("tipo");
    const parametros = document.getElementById("parametros");
    const formulaInput = document.getElementById("formula");
    const formulaDiv = document.getElementById("formulapersonalizada");
    const objeto = document.getElementById("objeto");

    tipoMovimiento.onchange = () => {
      const tipo = tipoMovimiento.value;
      parametros.innerHTML = "";
      formulaDiv.style.display = "none";

      if (tipo === "mru") {
        parametros.innerHTML = `
          <label>Velocidad (v):</label>
          <input type="number" id="velocidad" value="50">`;
      } else if (tipo === "mruv") {
        parametros.innerHTML = `
          <label>Velocidad inicial (v₀):</label>
          <input type="number" id="vo" value="0">
          <label>Aceleración (a):</label>
          <input type="number" id="a" value="10">`;
      } else if (tipo === "mrv" || tipo === "personalizado") {
        formulaDiv.style.display = "block";
      }
    };

    function iniciarSimulacion() {
      clearInterval(intervalo);
      tiempo = 0;
      datosPos = [], datosVel = [], datosAcel = [];
      objeto.style.left = "0px";
      if (graficaPos) graficaPos.destroy();
      if (graficaVel) graficaVel.destroy();
      if (graficaAcel) graficaAcel.destroy();

      const tipo = tipoMovimiento.value;
      const dt = 0.01;
      let expr = null;

      if (tipo === "mrv" || tipo === "personalizado") {
        try {
          expr = math.parse(formulaInput.value).compile();
        } catch (err) {
          alert("Error en la fórmula. Revísala.");
          return;
        }
      }

      intervalo = setInterval(() => {
        tiempo += 0.1;
        let x = 0, v = 0, a = 0;

        if (tipo === "mru") {
          const v0 = parseFloat(document.getElementById("velocidad").value);
          x = v0 * tiempo;
          v = v0;
          a = 0;
        } else if (tipo === "mruv") {
          const v0 = parseFloat(document.getElementById("vo").value);
          const acel = parseFloat(document.getElementById("a").value);
          x = v0 * tiempo + 0.5 * acel * tiempo ** 2;
          v = v0 + acel * tiempo;
          a = acel;
        } else if (expr) {
          try {
            const scope = { t: tiempo };
            const scope1 = { t: tiempo + dt };
            const scope2 = { t: tiempo + 2 * dt };
            const x0 = expr.evaluate(scope);
            const x1 = expr.evaluate(scope1);
            const x2 = expr.evaluate(scope2);
            x = x0;
            v = (x1 - x0) / dt;
            a = (x2 - 2 * x1 + x0) / (dt ** 2);
          } catch {
            clearInterval(intervalo);
            alert("Error evaluando la fórmula.");
            return;
          }
        }

        objeto.style.left = (x * 2) + "px";
        datosPos.push({ x: tiempo.toFixed(2), y: x });
        datosVel.push({ x: tiempo.toFixed(2), y: v });
        datosAcel.push({ x: tiempo.toFixed(2), y: a });

        if (tiempo >= 10) {
          clearInterval(intervalo);
          dibujarGraficas();
        }
      }, 100);
    }

    function dibujarGraficas() {
      const ctx1 = document.getElementById("graficaPos").getContext("2d");
      const ctx2 = document.getElementById("graficaVel").getContext("2d");
      const ctx3 = document.getElementById("graficaAcel").getContext("2d");

      graficaPos = new Chart(ctx1, {
        type: 'line',
        data: {
          datasets: [{
            label: 'Posición vs Tiempo',
            data: datosPos,
            borderColor: '#273c75',
            fill: false
          }]
        },
        options: {
          scales: {
            x: { title: { display: true, text: 'Tiempo (s)' } },
            y: { title: { display: true, text: 'Posición (m)' } }
          }
        }
      });

      graficaVel = new Chart(ctx2, {
        type: 'line',
        data: {
          datasets: [{
            label: 'Velocidad vs Tiempo',
            data: datosVel,
            borderColor: '#44bd32',
            fill: false
          }]
        },
        options: {
          scales: {
            x: { title: { display: true, text: 'Tiempo (s)' } },
            y: { title: { display: true, text: 'Velocidad (m/s)' } }
          }
        }
      });

      graficaAcel = new Chart(ctx3, {
        type: 'line',
        data: {
          datasets: [{
            label: 'Aceleración vs Tiempo',
            data: datosAcel,
            borderColor: '#e84118',
            fill: false
          }]
        },
        options: {
          scales: {
            x: { title: { display: true, text: 'Tiempo (s)' } },
            y: { title: { display: true, text: 'Aceleración (m/s²)' } }
          }
        }
      });
    }

    tipoMovimiento.dispatchEvent(new Event("change"));
  </script>
</body>
</html>
