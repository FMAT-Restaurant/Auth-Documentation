# Prompt Opcional: Sincronización de Modo Oscuro con el Shell

Si tu equipo desea que su microfrontend adapte sus colores automáticamente cuando el usuario activa o desactiva el **Modo Oscuro** en el Shell (`Auth-Frontend`), copia y pega el siguiente bloque en tu agente de IA:

---

````markdown
# PROMPT PARA AGENTE IA: Sincronizar Modo Oscuro con el Auth Shell

Actúa como un Ingeniero Frontend. Necesito que este microfrontend soporte el **Modo Oscuro** sincronizado con el Shell principal (`Auth-Frontend`) de **FMAT-Restaurant**.

### Cómo funciona el tema en el Shell
1. El Shell conmuta el atributo `data-theme="dark"` (o `"light"`) directamente en `document.documentElement` (`<html data-theme="dark">`).
2. Además, el Shell pasa la prop opcional `theme?: 'light' | 'dark'` al componente raíz `<App />` federado.

### Tareas a realizar en este repositorio
1. En `src/index.css` (o archivo global de estilos), define la paleta de variables CSS del sistema bajo `[data-theme="dark"]` (y asegura que los fondos, textos y bordes de tus componentes utilicen estas variables o clases condicionales):
   ```css
   [data-theme="dark"] {
     --color-primary: #E06A3B;
     --color-primary-hover: #F07C48;
     --color-accent: #FB923C;
     --color-primary-soft: rgba(224, 106, 59, 0.16);

     --color-ink: #FFFFFF;
     --color-text: #E8E3DF;
     --color-muted: #9C9288;
     --color-border: #2C2420;
     --color-canvas: #0A0908;
     --color-surface: #161311;

     --color-success: #22C55E;
     --bg-success: rgba(34, 197, 94, 0.15);
     --color-warning: #F59E0B;
     --bg-warning: rgba(245, 158, 11, 0.15);
     --color-danger: #EF4444;
     --bg-danger: rgba(239, 68, 68, 0.15);
   }
   ```
2. Si utilizas Tailwind CSS con `darkMode: 'class'` o `[data-theme="dark"]`, configura el selector para que responda a `[data-theme="dark"]` en `<html>` (o sincroniza la clase `.dark` cuando cambie la prop `theme`).
3. Asegúrate de que el contenedor raíz del módulo utilice `background-color: transparent` o `var(--color-canvas)` y `color: var(--color-text)` para integrarse visualmente sin parches blancos cuando el Shell esté en modo oscuro.
````
