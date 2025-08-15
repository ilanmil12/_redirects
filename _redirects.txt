import React, { useState, useEffect } from 'react';

// Main application component
export default function App() {
  // Estado para almacenar las respuestas del usuario
  const [answers, setAnswers] = useState({});
  // Estado para los puntajes e interpretaciones
  const [results, setResults] = useState({});
  // Estado para el informe generado por el LLM
  const [llmReport, setLlmReport] = useState(null);
  // Estado para el indicador de carga del LLM
  const [isLoading, setIsLoading] = useState(false);
  // Estado para los errores de la API
  const [error, setError] = useState(null);

  // Definición de las escalas, preguntas y opciones
  const scales = [
    {
      id: 'datos_paciente',
      title: 'Sección 1 – DATOS DEL PACIENTE',
      description: 'Proporcione la información demográfica y de identificación del paciente.',
      questions: [
        { id: 'nombre', text: 'Nombre completo', type: 'text', required: true },
        { id: 'edad', text: 'Edad', type: 'text' },
        { id: 'sexo', text: 'Sexo', type: 'radio', options: ['Hombre', 'Mujer', 'Prefiere no decirlo', 'Otro'] },
        { id: 'fecha', text: 'Fecha de evaluación', type: 'date' },
        { id: 'id', text: 'ID de Historia Clínica', type: 'text' },
      ],
    },
    {
      id: 'katz',
      title: 'Sección 2 – ÍNDICE DE KATZ (ABVD)',
      description: 'Objetivo: Identificar el grado de independencia en actividades básicas de la vida diaria (ABVD). Instrucciones: Marque la opción que mejor describa la situación.',
      questions: [
        { id: 'baño', text: 'Baño', type: 'radio', options: ['Independiente (1 punto)', 'Dependiente (0 puntos)'], points: [1, 0] },
        { id: 'vestido', text: 'Vestido', type: 'radio', options: ['Independiente (1 punto)', 'Dependiente (0 puntos)'], points: [1, 0] },
        { id: 'sanitario', text: 'Uso del sanitario', type: 'radio', options: ['Independiente (1 punto)', 'Dependiente (0 puntos)'], points: [1, 0] },
        { id: 'transferencias', text: 'Transferencias (cama/silla)', type: 'radio', options: ['Independiente (1 punto)', 'Dependiente (0 puntos)'], points: [1, 0] },
        { id: 'continencia', text: 'Continencia', type: 'radio', options: ['Continente (1 punto)', 'Incontinente (0 puntos)'], points: [1, 0] },
        { id: 'alimentacion', text: 'Alimentación', type: 'radio', options: ['Independiente (1 punto)', 'Dependiente (0 puntos)'], points: [1, 0] },
      ],
      calculateScore: (answers) => {
        const score = Object.values(answers).reduce((sum, val) => sum + (val !== undefined ? val : 0), 0);
        let interpretation = '';
        switch (score) {
          case 6: interpretation = 'A: Independencia total'; break;
          case 5: interpretation = 'B: Independencia excepto en 1 actividad'; break;
          case 4: interpretation = 'C: Dependiente en baño y 1 actividad adicional'; break;
          case 3: interpretation = 'D: Dependiente en baño, vestido y 1 actividad adicional'; break;
          case 2: interpretation = 'E: Dependiente en baño, vestido, uso del sanitario y 1 actividad adicional'; break;
          case 1: interpretation = 'F: Dependiente en baño, vestido, uso del sanitario, transferencias y 1 actividad adicional'; break;
          case 0: interpretation = 'G: Dependiente en las 6 ABVD'; break;
          default:
            interpretation = 'H: Dependiente en 2 actividades que no encajan en C, D, E o F';
            // Simple check for H category
            const dependentActivities = Object.keys(answers).filter(key => answers[key] === 0);
            if (dependentActivities.length === 2 && !['baño', 'vestido', 'sanitario', 'transferencias'].includes(dependentActivities.join())) {
              interpretation = 'H: Dependiente en 2 actividades que no encajan en C, D, E o F';
            }
        }
        return { score, interpretation };
      },
    },
    {
      id: 'barthel',
      title: 'Sección 3 – ÍNDICE DE BARTHEL (ABVD)',
      description: 'Objetivo: Cuantificar la independencia en diez actividades básicas. Instrucciones: Seleccione la opción que mejor describa la capacidad.',
      questions: [
        { id: 'barthel_baño', text: 'Baño / Ducha', type: 'radio', options: ['Independiente (5 puntos)', 'Dependiente (0 puntos)'], points: [5, 0] },
        { id: 'barthel_vestido', text: 'Vestido', type: 'radio', options: ['Independiente (10)', 'Necesita ayuda parcial (5)', 'Dependiente (0)'], points: [10, 5, 0] },
        { id: 'barthel_aseo', text: 'Aseo personal', type: 'radio', options: ['Independiente (5)', 'Dependiente (0)'], points: [5, 0] },
        { id: 'barthel_sanitario', text: 'Uso del retrete', type: 'radio', options: ['Independiente (10)', 'Necesita ayuda (5)', 'Dependiente (0)'], points: [10, 5, 0] },
        { id: 'barthel_escaleras', text: 'Uso de escaleras', type: 'radio', options: ['Independiente (10)', 'Necesita ayuda (5)', 'Dependiente (0)'], points: [10, 5, 0] },
        { id: 'barthel_traslado', text: 'Traslado cama–silla', type: 'radio', options: ['Independiente (15)', 'Ayuda mínima (10)', 'Gran ayuda (5)', 'Dependiente (0)'], points: [15, 10, 5, 0] },
        { id: 'barthel_desplazamiento', text: 'Desplazamiento', type: 'radio', options: ['Independiente (15)', 'Ayuda (10)', 'Independiente en silla de ruedas (5)', 'Dependiente (0)'], points: [15, 10, 5, 0] },
        { id: 'barthel_orina', text: 'Control de orina', type: 'radio', options: ['Continente (10)', 'Incontinencia ocasional (5)', 'Incontinente (0)'], points: [10, 5, 0] },
        { id: 'barthel_heces', text: 'Control de heces', type: 'radio', options: ['Continente (10)', 'Incontinencia ocasional (5)', 'Incontinente (0)'], points: [10, 5, 0] },
        { id: 'barthel_alimentacion', text: 'Alimentación', type: 'radio', options: ['Independiente (10)', 'Necesita ayuda (5)', 'Dependiente (0)'], points: [10, 5, 0] },
      ],
      calculateScore: (answers) => {
        const score = Object.values(answers).reduce((sum, val) => sum + (val !== undefined ? val : 0), 0);
        let interpretation = '';
        if (score <= 40) interpretation = '0–40: Dependencia grave';
        else if (score >= 45 && score <= 60) interpretation = '45–60: Dependencia moderada';
        else if (score >= 65 && score <= 100) interpretation = '65–100: Dependencia leve / Independencia';
        return { score, interpretation };
      },
    },
    {
      id: 'lawton',
      title: 'Sección 4 – ESCALA DE LAWTON (AIVD)',
      description: 'Objetivo: Evaluar la capacidad en actividades más complejas (instrumentales) necesarias para la vida independiente. Puntaje máximo = 8 puntos (1 punto por actividad independiente).',
      questions: [
        { id: 'lawton_telefono', text: 'Uso del teléfono', type: 'radio', options: ['Independiente (1 punto)', 'Dependiente (0)'], points: [1, 0] },
        { id: 'lawton_transporte', text: 'Transporte', type: 'radio', options: ['Independiente (1)', 'Dependiente (0)'], points: [1, 0] },
        { id: 'lawton_medicamentos', text: 'Medicamentos', type: 'radio', options: ['Independiente (1)', 'Dependiente (0)'], points: [1, 0] },
        { id: 'lawton_finanzas', text: 'Finanzas', type: 'radio', options: ['Independiente (1)', 'Dependiente (0)'], points: [1, 0] },
        { id: 'lawton_compras', text: 'Compras', type: 'radio', options: ['Independiente (1)', 'Dependiente (0)'], points: [1, 0] },
        { id: 'lawton_cocina', text: 'Cocina', type: 'radio', options: ['Independiente (1)', 'Dependiente (0)'], points: [1, 0] },
        { id: 'lawton_hogar', text: 'Cuidado del hogar', type: 'radio', options: ['Independiente (1)', 'Dependiente (0)'], points: [1, 0] },
        { id: 'lawton_lavanderia', text: 'Lavandería', type: 'radio', options: ['Independiente (1)', 'Dependiente (0)'], points: [1, 0] },
      ],
      calculateScore: (answers) => {
        const score = Object.values(answers).reduce((sum, val) => sum + (val !== undefined ? val : 0), 0);
        let interpretation = '';
        if (score >= 0 && score <= 2) interpretation = '0–2: Dependencia total';
        else if (score >= 3 && score <= 5) interpretation = '3–5: Dependencia moderada';
        else if (score >= 6 && score <= 8) interpretation = '6–8: Independencia en AIVD';
        return { score, interpretation };
      },
    },
    {
      id: 'norton',
      title: 'Sección 5 – ESCALA DE NORTON – RIESGO DE ÚLCERAS POR PRESIÓN',
      description: 'Objetivo: Valorar el riesgo de úlceras por presión. Cada criterio se puntúa de 1 a 4 puntos (4 = mejor estado).',
      questions: [
        { id: 'norton_fisico', text: 'Estado físico', type: 'radio', options: ['4 puntos – Bueno', '3 puntos – Débil', '2 puntos – Malo', '1 punto – Muy mala'], points: [4, 3, 2, 1] },
        { id: 'norton_mental', text: 'Estado mental', type: 'radio', options: ['4 – Alerta', '3 – Apático', '2 – Confuso', '1 – Estuporoso'], points: [4, 3, 2, 1] },
        { id: 'norton_actividad', text: 'Actividad', type: 'radio', options: ['4 – Camina', '3 – Camina con ayuda', '2 – En silla de ruedas', '1 – En cama'], points: [4, 3, 2, 1] },
        { id: 'norton_movilidad', text: 'Movilidad', type: 'radio', options: ['4 – Completa', '3 – Limitada ligeramente', '2 – Muy limitada', '1 – Inmóvil'], points: [4, 3, 2, 1] },
        { id: 'norton_incontinencia', text: 'Incontinencia', type: 'radio', options: ['4 – No hay', '3 – Ocasional', '2 – Usualmente urinaria', '1 – Doble incontinencia'], points: [4, 3, 2, 1] },
      ],
      calculateScore: (answers) => {
        const score = Object.values(answers).reduce((sum, val) => sum + (val !== undefined ? val : 0), 0);
        let interpretation = '';
        if (score >= 17) interpretation = '≥17: Bajo riesgo';
        else if (score >= 12 && score <= 16) interpretation = '12–16: Riesgo moderado';
        else if (score < 12) interpretation = '<12: Riesgo alto';
        return { score, interpretation };
      },
    },
    {
      id: 'sarcf',
      title: 'Sección 6 – SARC‑F (CUESTIONARIO DE SARCOPENIA)',
      description: 'Objetivo: Detectar de forma rápida la sarcopenia. (0 = sin dificultad, 2 = gran dificultad/invalidez)',
      questions: [
        { id: 'sarcf_fuerza', text: 'Fuerza al levantar objetos de 5 kg', type: 'radio', options: ['0 puntos – Sin dificultad', '1 punto – Alguna dificultad', '2 puntos – Mucha dificultad o incapaz'], points: [0, 1, 2] },
        { id: 'sarcf_caminar', text: 'Asistencia para caminar', type: 'radio', options: ['0 – No necesita ayuda', '1 – Usa bastón o soporte', '2 – Usa andador o incapaz de caminar'], points: [0, 1, 2] },
        { id: 'sarcf_silla', text: 'Levantarse de la silla', type: 'radio', options: ['0 – Puede levantarse sin dificultad', '1 – Alguna dificultad', '2 – Mucha dificultad o incapaz'], points: [0, 1, 2] },
        { id: 'sarcf_escaleras', text: 'Subir escaleras', type: 'radio', options: ['0 – Sin dificultad', '1 – Alguna dificultad', '2 – Mucha dificultad o incapaz'], points: [0, 1, 2] },
        { id: 'sarcf_caidas', text: 'Caídas en el último año', type: 'radio', options: ['0 – Ninguna', '1 – 1–3 caídas', '2 – 4 o más caídas'], points: [0, 1, 2] },
      ],
      calculateScore: (answers) => {
        const score = Object.values(answers).reduce((sum, val) => sum + (val !== undefined ? val : 0), 0);
        let interpretation = '';
        if (score >= 0 && score <= 3) interpretation = '0–3: Bajo riesgo de sarcopenia';
        else if (score >= 4) interpretation = '≥4: Probable sarcopenia';
        return { score, interpretation };
      },
    },
    {
      id: 'mmse',
      title: 'Sección 7 – MINI‑MENTAL STATE EXAMINATION (MMSE)',
      description: 'Objetivo: Evaluar la función cognitiva general. Instrucciones: Registre el puntaje obtenido en cada dominio (el evaluador administra las tareas al paciente).',
      questions: [
        { id: 'mmse_orientacion_temporal', text: 'Orientación temporal (0–5)', type: 'number', min: 0, max: 5 },
        { id: 'mmse_orientacion_espacial', text: 'Orientación espacial (0–5)', type: 'number', min: 0, max: 5 },
        { id: 'mmse_registro', text: 'Registro (memoria inmediata) (0–3)', type: 'number', min: 0, max: 3 },
        { id: 'mmse_atencion', text: 'Atención y cálculo (0–5)', type: 'number', min: 0, max: 5 },
        { id: 'mmse_memoria_diferida', text: 'Memoria diferida (0–3)', type: 'number', min: 0, max: 3 },
        { id: 'mmse_lenguaje', text: 'Lenguaje y praxis (0–9)', type: 'number', min: 0, max: 9 },
      ],
      calculateScore: (answers) => {
        const score = Object.values(answers).reduce((sum, val) => sum + (val !== undefined ? parseInt(val, 10) : 0), 0);
        let interpretation = '';
        if (score >= 24) interpretation = '≥24: Normal';
        else if (score >= 18 && score <= 23) interpretation = '18–23: Deterioro cognitivo leve/moderado';
        else if (score <= 17) interpretation = '≤17: Deterioro cognitivo grave';
        return { score, interpretation };
      },
    },
    {
      id: 'marcha',
      title: 'Sección 8 – VELOCIDAD DE MARCHA (4 METROS)',
      description: 'Objetivo: Medir la capacidad funcional y predecir riesgo de eventos adversos. Instrucciones: Anote el tiempo y la aplicación calculará la velocidad y su interpretación.',
      questions: [
        { id: 'marcha_tiempo', text: 'Tiempo en segundos', type: 'number' },
      ],
      calculateScore: (answers) => {
        const tiempo = parseFloat(answers.marcha_tiempo);
        const velocidad = tiempo > 0 ? (4 / tiempo).toFixed(2) : 0;
        let interpretation = '';
        if (velocidad >= 1.0) interpretation = '1,0 m/s: Normal';
        else if (velocidad >= 0.8 && velocidad < 1.0) interpretation = '0,8–1,0 m/s: Leve disminución del desempeño';
        else if (velocidad > 0 && velocidad < 0.8) interpretation = '<0,8 m/s: Alta probabilidad de sarcopenia/desenlaces adversos';
        return { score: velocidad, interpretation };
      },
    },
    {
      id: 'maltrato',
      title: 'Sección 9 – ESCALA GERIÁTRICA DE MALTRATO AL ADULTO MAYOR',
      description: 'Objetivo: Detectar situaciones de abuso. Instrucciones: Marque “Sí” o “No” según corresponda.',
      questions: [
        { id: 'maltrato_1', text: '¿Le han golpeado?', type: 'radio', options: ['Sí', 'No'], points: [1, 0] },
        { id: 'maltrato_2', text: '¿Le han dado puñetazos o patadas?', type: 'radio', options: ['Sí', 'No'], points: [1, 0] },
        { id: 'maltrato_3', text: '¿Le han empujado o jalado el pelo?', type: 'radio', options: ['Sí', 'No'], points: [1, 0] },
        { id: 'maltrato_4', text: '¿Le han humillado o se han burlado de usted?', type: 'radio', options: ['Sí', 'No'], points: [1, 0] },
        { id: 'maltrato_5', text: '¿Le han aislado o corrido de la casa?', type: 'radio', options: ['Sí', 'No'], points: [1, 0] },
        { id: 'maltrato_6', text: '¿Le han hecho sentir miedo?', type: 'radio', options: ['Sí', 'No'], points: [1, 0] },
        { id: 'maltrato_7', text: '¿Le han prohibido salir o que la visiten?', type: 'radio', options: ['Sí', 'No'], points: [1, 0] },
        { id: 'maltrato_8', text: '¿Le han negado ropa o medicamentos necesarios?', type: 'radio', options: ['Sí', 'No'], points: [1, 0] },
        { id: 'maltrato_9', text: '¿Le han quitado dinero o bienes?', type: 'radio', options: ['Sí', 'No'], points: [1, 0] },
        { id: 'maltrato_10', text: '¿Le han presionado para renunciar a sus propiedades?', type: 'radio', options: ['Sí', 'No'], points: [1, 0] },
        { id: 'maltrato_11', text: '¿Le han exigido relaciones sexuales contra su voluntad?', type: 'radio', options: ['Sí', 'No'], points: [1, 0] },
      ],
      calculateScore: (answers) => {
        const score = Object.values(answers).filter(val => val === 1).length;
        let interpretation = '';
        if (score <= 1) interpretation = '0–1: No sugiere maltrato';
        else interpretation = '≥2: Maltrato probable';
        return { score, interpretation };
      },
    },
    {
      id: 'dormir',
      title: 'Sección 10 – IDENTIFICACIÓN DE TRASTORNOS DEL DORMIR',
      description: 'Objetivo: Identificar insomnio, somnolencia diurna, apnea obstructiva y nocturia. Instrucciones: Marque “Sí” o “No” según corresponda.',
      questions: [
        { id: 'insomnio', text: '¿Tiene dificultad para conciliar el sueño?', type: 'radio', options: ['Sí', 'No'] },
        { id: 'insomnio_mantener', text: '¿Tiene dificultad para mantenerse dormido(a)?', type: 'radio', options: ['Sí', 'No'] },
        { id: 'somnolencia', text: '¿Tiene somnolencia diurna excesiva?', type: 'radio', options: ['Sí', 'No'] },
        { id: 'ronquidos', text: '¿Tiene ronquidos intensos al dormir?', type: 'radio', options: ['Sí', 'No'] },
        { id: 'nocturia', text: '¿Presenta tres o más micciones durante la noche?', type: 'radio', options: ['Sí', 'No'] },
      ],
      calculateScore: (answers) => {
        let interpretation = [];
        if (answers.insomnio === 'Sí' || answers.insomnio_mantener === 'Sí') interpretation.push('Probable insomnio');
        if (answers.somnolencia === 'Sí') interpretation.push('Probable somnolencia diurna');
        if (answers.ronquidos === 'Sí') interpretation.push('Probable apnea obstructiva del sueño');
        if (answers.nocturia === 'Sí') interpretation.push('Probable nocturia');
        if (interpretation.length === 0) interpretation.push('Ningún trastorno identificado');
        return { score: null, interpretation: interpretation.join(', ') };
      },
    },
    {
      id: 'zarit',
      title: 'Sección 11 – TEST DE SOBRECARGA DEL CUIDADOR (ZARIT)',
      description: 'Objetivo: Medir la carga percibida por el cuidador. Instrucciones: Para cada afirmación, seleccione con qué frecuencia se siente así (0 = nunca, 4 = casi siempre).',
      questions: [
        { id: 'zarit_1', text: '¿Piensa que su familiar le pide más ayuda de la que realmente necesita?', type: 'radio', options: ['0', '1', '2', '3', '4'], points: [0, 1, 2, 3, 4] },
        { id: 'zarit_2', text: '¿Piensa que debido al tiempo que dedica a su familiar no tiene suficiente tiempo para usted?', type: 'radio', options: ['0', '1', '2', '3', '4'], points: [0, 1, 2, 3, 4] },
        { id: 'zarit_3', text: '¿Se siente agobiado al compatibilizar el cuidado con otras responsabilidades?', type: 'radio', options: ['0', '1', '2', '3', '4'], points: [0, 1, 2, 3, 4] },
        { id: 'zarit_4', text: '¿Siente vergüenza por la conducta de su familiar?', type: 'radio', options: ['0', '1', '2', '3', '4'], points: [0, 1, 2, 3, 4] },
        { id: 'zarit_5', text: '¿Se siente enfadado cuando está cerca de su familiar?', type: 'radio', options: ['0', '1', '2', '3', '4'], points: [0, 1, 2, 3, 4] },
        { id: 'zarit_6', text: '¿Piensa que el cuidado afecta negativamente su relación con otros miembros de la familia?', type: 'radio', options: ['0', '1', '2', '3', '4'], points: [0, 1, 2, 3, 4] },
        { id: 'zarit_7', text: '¿Tiene miedo por el futuro de su familiar?', type: 'radio', options: ['0', '1', '2', '3', '4'], points: [0, 1, 2, 3, 4] },
        { id: 'zarit_8', text: '¿Piensa que su familiar depende totalmente de usted?', type: 'radio', options: ['0', '1', '2', '3', '4'], points: [0, 1, 2, 3, 4] },
        { id: 'zarit_9', text: '¿Se siente tenso cuando está cerca de su familiar?', type: 'radio', options: ['0', '1', '2', '3', '4'], points: [0, 1, 2, 3, 4] },
        { id: 'zarit_10', text: '¿Piensa que su salud ha empeorado debido al cuidado?', type: 'radio', options: ['0', '1', '2', '3', '4'], points: [0, 1, 2, 3, 4] },
        { id: 'zarit_11', text: '¿Piensa que no tiene tanta intimidad como le gustaría?', type: 'radio', options: ['0', '1', '2', '3', '4'], points: [0, 1, 2, 3, 4] },
        { id: 'zarit_12', text: '¿Piensa que su vida social se ha visto afectada negativamente?', type: 'radio', options: ['0', '1', '2', '3', '4'], points: [0, 1, 2, 3, 4] },
        { id: 'zarit_13', text: '¿Se siente incómodo por distanciarse de sus amistades?', type: 'radio', options: ['0', '1', '2', '3', '4'], points: [0, 1, 2, 3, 4] },
        { id: 'zarit_14', text: '¿Piensa que su familiar le considera la única persona que le puede cuidar?', type: 'radio', options: ['0', '1', '2', '3', '4'], points: [0, 1, 2, 3, 4] },
        { id: 'zarit_15', text: '¿Piensa que no tiene suficientes ingresos para los gastos del cuidado?', type: 'radio', options: ['0', '1', '2', '3', '4'], points: [0, 1, 2, 3, 4] },
        { id: 'zarit_16', text: '¿Piensa que no será capaz de cuidar a su familiar por mucho más tiempo?', type: 'radio', options: ['0', '1', '2', '3', '4'], points: [0, 1, 2, 3, 4] },
        { id: 'zarit_17', text: '¿Siente que ha perdido el control de su vida desde que comenzó la enfermedad de su familiar?', type: 'radio', options: ['0', '1', '2', '3', '4'], points: [0, 1, 2, 3, 4] },
        { id: 'zarit_18', text: '¿Desearía poder dejar el cuidado de su familiar a otra persona?', type: 'radio', options: ['0', '1', '2', '3', '4'], points: [0, 1, 2, 3, 4] },
        { id: 'zarit_19', text: '¿Se siente indeciso sobre qué hacer con su familiar?', type: 'radio', options: ['0', '1', '2', '3', '4'], points: [0, 1, 2, 3, 4] },
        { id: 'zarit_20', text: '¿Piensa que debería hacer más por su familiar?', type: 'radio', options: ['0', '1', '2', '3', '4'], points: [0, 1, 2, 3, 4] },
        { id: 'zarit_21', text: '¿Piensa que podría cuidar mejor a su familiar?', type: 'radio', options: ['0', '1', '2', '3', '4'], points: [0, 1, 2, 3, 4] },
        { id: 'zarit_22', text: 'Globalmente, ¿qué grado de carga experimenta al cuidar a su familiar?', type: 'radio', options: ['0', '1', '2', '3', '4'], points: [0, 1, 2, 3, 4] },
      ],
      calculateScore: (answers) => {
        const score = Object.values(answers).reduce((sum, val) => sum + (val !== undefined ? val : 0), 0);
        let interpretation = '';
        if (score < 47) interpretation = '<47: Sin sobrecarga';
        else if (score >= 47 && score <= 55) interpretation = '47–55: Sobrecarga leve';
        else if (score > 55) interpretation = '55: Sobrecarga intensa';
        return { score, interpretation };
      },
    },
    {
      id: 'sppb',
      title: 'Sección 12 – BATERÍA CORTA DE RENDIMIENTO FÍSICO (SPPB)',
      description: 'Objetivo: Evaluar la función física mediante tres pruebas. Instrucciones: Asigne el puntaje de 0 a 4 a cada prueba según tablas validadas.',
      questions: [
        { id: 'sppb_equilibrio', text: 'Puntaje de prueba de equilibrio (0–4)', type: 'number', min: 0, max: 4 },
        { id: 'sppb_marcha', text: 'Puntaje de velocidad de marcha (0–4)', type: 'number', min: 0, max: 4 },
        { id: 'sppb_silla', text: 'Puntaje de levantarse de la silla 5 veces (0–4)', type: 'number', min: 0, max: 4 },
      ],
      calculateScore: (answers) => {
        const score = Object.values(answers).reduce((sum, val) => sum + (val !== undefined ? parseInt(val, 10) : 0), 0);
        let interpretation = '';
        if (score >= 0 && score <= 3) interpretation = '0–3: Desempeño muy bajo';
        else if (score >= 4 && score <= 6) interpretation = '4–6: Desempeño bajo (fragilidad probable)';
        else if (score >= 7 && score <= 9) interpretation = '7–9: Desempeño moderado';
        else if (score >= 10 && score <= 12) interpretation = '10–12: Desempeño alto';
        return { score, interpretation };
      },
    },
    {
      id: 'mini_cog',
      title: 'Sección 13 – MINI‑COG',
      description: 'Objetivo: Cribado rápido de deterioro cognitivo. Instrucciones: El evaluador pide al paciente repetir tres palabras, dibujar un reloj y repetirlas al final.',
      questions: [
        { id: 'mini_cog_palabras', text: 'Número de palabras recordadas (0–3)', type: 'number', min: 0, max: 3 },
        { id: 'mini_cog_reloj', text: 'Reloj dibujado correctamente', type: 'radio', options: ['Sí', 'No'] },
      ],
      calculateScore: (answers) => {
        const score = answers.mini_cog_palabras !== undefined ? parseInt(answers.mini_cog_palabras, 10) : 0;
        let interpretation = '';
        if (score >= 0 && score <= 2) interpretation = '0–2 puntos: Posible deterioro cognitivo';
        else if (score === 3) interpretation = '3 puntos: Función cognitiva normal';
        return { score, interpretation };
      },
    },
    {
      id: 'moca',
      title: 'Sección 14 – MOCA (MONTREAL COGNITIVE ASSESSMENT)',
      description: 'Objetivo: Evaluar diferentes dominios cognitivos. Instrucciones: Ingrese el puntaje de cada dominio según la ejecución del paciente (0 = peor, máximo según dominio).',
      questions: [
        { id: 'moca_visuoespacial', text: 'Visuoespacial/Ejecutivo (0–5)', type: 'number', min: 0, max: 5 },
        { id: 'moca_denominacion', text: 'Denominación (0–3)', type: 'number', min: 0, max: 3 },
        { id: 'moca_memoria', text: 'Memoria (0–5)', type: 'number', min: 0, max: 5 },
        { id: 'moca_atencion', text: 'Atención (0–6)', type: 'number', min: 0, max: 6 },
        { id: 'moca_lenguaje', text: 'Lenguaje (0–3)', type: 'number', min: 0, max: 3 },
        { id: 'moca_abstraccion', text: 'Abstracción (0–2)', type: 'number', min: 0, max: 2 },
        { id: 'moca_orientacion', text: 'Orientación (0–6)', type: 'number', min: 0, max: 6 },
      ],
      calculateScore: (answers) => {
        const score = Object.values(answers).reduce((sum, val) => sum + (val !== undefined ? parseInt(val, 10) : 0), 0);
        let interpretation = '';
        if (score >= 26 && score <= 30) interpretation = '26–30: Cognición normal';
        else if (score >= 18 && score <= 25) interpretation = '18–25: Deterioro cognitivo leve';
        else if (score < 18) interpretation = '<18: Deterioro cognitivo moderado/severo';
        return { score, interpretation };
      },
    },
  ];

  // Manejador de cambios para actualizar las respuestas del usuario
  const handleAnswerChange = (scaleId, questionId, value) => {
    setAnswers(prevAnswers => ({
      ...prevAnswers,
      [scaleId]: {
        ...prevAnswers[scaleId],
        [questionId]: value,
      },
    }));
  };

  // Efecto para recalcular los puntajes cada vez que las respuestas cambian
  useEffect(() => {
    const newResults = {};
    scales.filter(s => s.calculateScore).forEach(scale => {
      const scaleAnswers = answers[scale.id] || {};
      newResults[scale.id] = scale.calculateScore(scaleAnswers);
    });
    setResults(newResults);
  }, [answers]);

  // Función para llamar a la API de Gemini y generar el reporte
  const handleGenerateReport = async () => {
    setIsLoading(true);
    setLlmReport(null);
    setError(null);

    // Formatear los resultados para el prompt del LLM
    const formattedResults = Object.keys(results).map(key => {
      const scale = scales.find(s => s.id === key);
      return `
        Escala: ${scale.title}
        Puntaje: ${results[key].score !== null ? results[key].score : 'N/A'}
        Interpretación: ${results[key].interpretation}
      `;
    }).join('\n');

    const prompt = `
      Genera un reporte clínico conciso basado en las siguientes evaluaciones geriátricas.
      El reporte debe incluir un resumen de los hallazgos principales y una lista de recomendaciones sugeridas.
      Responde SOLO con un objeto JSON en el siguiente formato:
      {
        "resumen": "string, resumen de los hallazgos clínicos",
        "recomendaciones": "string[], lista de recomendaciones clínicas"
      }

      A continuación, los resultados de la evaluación:
      ${formattedResults}
    `;

    const payload = {
      contents: [{
        parts: [{ text: prompt }]
      }],
      generationConfig: {
        responseMimeType: 'application/json',
        responseSchema: {
          type: 'OBJECT',
          properties: {
            "resumen": { "type": "STRING" },
            "recomendaciones": { "type": "ARRAY", "items": { "type": "STRING" } }
          },
          propertyOrdering: ["resumen", "recomendaciones"]
        }
      }
    };

    const apiKey = "";
    const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-05-20:generateContent?key=${apiKey}`;

    let retryCount = 0;
    const maxRetries = 3;

    while (retryCount < maxRetries) {
      try {
        const response = await fetch(apiUrl, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(payload)
        });

        if (!response.ok) {
          throw new Error(`Error en la respuesta de la API: ${response.status} ${response.statusText}`);
        }

        const result = await response.json();
        const json = result?.candidates?.[0]?.content?.parts?.[0]?.text;

        if (json) {
          const parsedJson = JSON.parse(json);
          setLlmReport(parsedJson);
          break; // Salir del bucle si es exitoso
        } else {
          throw new Error("Respuesta de la API inesperada o vacía.");
        }
      } catch (e) {
        console.error("Error al llamar a la API:", e);
        if (e.message.includes("429") && retryCount < maxRetries - 1) {
          const delay = Math.pow(2, retryCount) * 1000;
          console.log(`Reintentando en ${delay / 1000} segundos...`);
          await new Promise(res => setTimeout(res, delay));
          retryCount++;
        } else {
          setError(`No se pudo generar el reporte. Por favor, intente de nuevo.`);
          break;
        }
      } finally {
        setIsLoading(false);
      }
    }
  };

  // Componente para renderizar cada sección del formulario
  const FormSection = ({ scale }) => {
    const scaleAnswers = answers[scale.id] || {};
    const scaleResults = results[scale.id] || {};

    return (
      <div className="bg-white dark:bg-gray-800 p-6 rounded-xl shadow-lg border border-gray-200 dark:border-gray-700 mb-8">
        <h2 className="text-2xl font-semibold mb-2 text-blue-700 dark:text-blue-300">{scale.title}</h2>
        <p className="text-gray-600 dark:text-gray-400 mb-4 whitespace-pre-line">{scale.description}</p>
        
        {scale.questions.map((q) => (
          <div key={q.id} className="mb-4">
            <label className="block text-lg font-medium text-gray-800 dark:text-gray-200 mb-2">
              {q.text} {q.required && <span className="text-red-500">*</span>}
            </label>
            {q.type === 'text' && (
              <input
                type="text"
                className="w-full p-2 border border-gray-300 dark:border-gray-600 rounded-lg bg-gray-50 dark:bg-gray-700 text-gray-900 dark:text-gray-100 focus:outline-none focus:ring-2 focus:ring-blue-500"
                onChange={(e) => handleAnswerChange(scale.id, q.id, e.target.value)}
                required={q.required}
              />
            )}
            {q.type === 'date' && (
              <input
                type="date"
                className="w-full p-2 border border-gray-300 dark:border-gray-600 rounded-lg bg-gray-50 dark:bg-gray-700 text-gray-900 dark:text-gray-100 focus:outline-none focus:ring-2 focus:ring-blue-500"
                onChange={(e) => handleAnswerChange(scale.id, q.id, e.target.value)}
              />
            )}
            {q.type === 'number' && (
              <input
                type="number"
                min={q.min}
                max={q.max}
                className="w-full p-2 border border-gray-300 dark:border-gray-600 rounded-lg bg-gray-50 dark:bg-gray-700 text-gray-900 dark:text-gray-100 focus:outline-none focus:ring-2 focus:ring-blue-500"
                onChange={(e) => handleAnswerChange(scale.id, q.id, e.target.value)}
              />
            )}
            {q.type === 'radio' && (
              <div className="space-y-2">
                {q.options.map((option, index) => (
                  <label key={index} className="flex items-center space-x-2">
                    <input
                      type="radio"
                      name={`${scale.id}-${q.id}`}
                      value={option}
                      onChange={() => handleAnswerChange(scale.id, q.id, q.points ? q.points[index] : option)}
                      className="form-radio h-5 w-5 text-blue-600 dark:text-blue-400 focus:ring-blue-500"
                    />
                    <span className="text-gray-700 dark:text-gray-300">{option}</span>
                  </label>
                ))}
              </div>
            )}
          </div>
        ))}

        {scale.calculateScore && (
          <div className="mt-6 p-4 bg-blue-50 dark:bg-blue-950 rounded-lg border-l-4 border-blue-500 dark:border-blue-400 shadow-inner">
            <h3 className="text-xl font-bold text-blue-800 dark:text-blue-200">Resultados</h3>
            {scale.id === 'marcha' ? (
              <>
                <p className="mt-2 text-blue-700 dark:text-blue-300">
                  <span className="font-semibold">Velocidad:</span> {scaleResults.score !== undefined ? `${scaleResults.score} m/s` : 'N/A'}
                </p>
                <p className="mt-1 text-blue-700 dark:text-blue-300">
                  <span className="font-semibold">Interpretación:</span> {scaleResults.interpretation}
                </p>
              </>
            ) : (
              <>
                <p className="mt-2 text-blue-700 dark:text-blue-300">
                  <span className="font-semibold">Puntaje total:</span> {scaleResults.score !== undefined ? scaleResults.score : 'N/A'}
                </p>
                <p className="mt-1 text-blue-700 dark:text-blue-300">
                  <span className="font-semibold">Interpretación:</span> {scaleResults.interpretation}
                </p>
              </>
            )}
          </div>
        )}
      </div>
    );
  };

  return (
    <div className="min-h-screen bg-gray-100 dark:bg-gray-900 text-gray-800 dark:text-gray-200 p-4 sm:p-8">
      <div className="max-w-4xl mx-auto">
        <header className="text-center mb-8">
          <h1 className="text-4xl font-extrabold text-blue-900 dark:text-blue-100 mb-2">Evaluador Geriátrico</h1>
          <p className="text-lg text-gray-600 dark:text-gray-400">Selecciona las respuestas para cada escala y obtén los puntajes e interpretaciones en tiempo real.</p>
        </header>

        <main>
          {scales.map((scale) => (
            <FormSection key={scale.id} scale={scale} />
          ))}

          <div className="flex justify-center mt-8">
            <button
              onClick={handleGenerateReport}
              disabled={isLoading}
              className={`flex items-center justify-center px-6 py-3 rounded-lg font-bold text-white transition-colors duration-300 ${
                isLoading ? 'bg-gray-500 cursor-not-allowed' : 'bg-green-600 hover:bg-green-700'
              }`}
            >
              {isLoading ? (
                <>
                  <svg className="animate-spin -ml-1 mr-3 h-5 w-5 text-white" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                    <circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4"></circle>
                    <path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
                  </svg>
                  Generando...
                </>
              ) : (
                <>Generar Reporte ✨</>
              )}
            </button>
          </div>

          {llmReport && (
            <div className="mt-8 p-6 bg-blue-50 dark:bg-blue-900 rounded-xl shadow-lg border border-blue-200 dark:border-blue-700">
              <h2 className="text-3xl font-extrabold text-blue-800 dark:text-blue-200 mb-4">Reporte Clínico Generado</h2>
              <div className="space-y-4">
                <p className="text-lg font-semibold text-gray-800 dark:text-gray-200">Resumen de Hallazgos:</p>
                <p className="text-gray-700 dark:text-gray-300 whitespace-pre-line">{llmReport.resumen}</p>
                <p className="text-lg font-semibold text-gray-800 dark:text-gray-200">Recomendaciones Sugeridas:</p>
                <ul className="list-disc list-inside space-y-2 text-gray-700 dark:text-gray-300">
                  {llmReport.recomendaciones.map((rec, index) => (
                    <li key={index}>{rec}</li>
                  ))}
                </ul>
              </div>
            </div>
          )}

          {error && (
            <div className="mt-8 p-4 bg-red-100 dark:bg-red-950 text-red-700 dark:text-red-300 rounded-lg border border-red-200 dark:border-red-700 text-center">
              <p>{error}</p>
            </div>
          )}
        </main>
      </div>
    </div>
  );
}
