# Technical Guide PDF Pipeline Specification

## Purpose

Define los requisitos de la documentación técnica en Markdown, maquetación visual HTML y compilación desatendida a PDF profesional estructurado con soporte de marcadores e identidad visual homogénea.

## Requirements

### Requirement: Estructura y formato de la guía técnica en Markdown
El repositorio SHALL contener la guía completa en formato Markdown con índice navegable, diagramas de arquitectura en texto claro, bloques de comandos con sintaxis resaltada y cajas de notas pedagógicas.

#### Scenario: Lectura y navegación del documento fuente
- **WHEN** un usuario consulta el archivo `docs/k8s-postgresql-statefulset.md`
- **THEN** encuentra una estructura capitular progresiva con explicaciones conceptuales, comandos reproducibles y salidas de consola reales

### Requirement: Generación desatendida de HTML y PDF profesional
El sistema SHALL proveer un script en Python que transforme la guía técnica en un documento HTML maquetado e invoque Chrome headless para generar un PDF de calidad editorial con marcadores de navegación.

#### Scenario: Compilación del documento PDF final
- **WHEN** se ejecuta el script `scripts/generate_html_pdf.py` en un entorno con Google Chrome o Chromium disponible
- **THEN** se genera el archivo `k8s-postgresql-statefulset.pdf` con portada formal, tipografías Inter y JetBrains Mono, numeración de páginas y árbol de marcadores en el visor PDF
