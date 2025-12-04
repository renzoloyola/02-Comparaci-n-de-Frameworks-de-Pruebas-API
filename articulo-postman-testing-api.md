# Dominando las Pruebas de API con Postman: Ejemplos del Mundo Real

## Introducción

En el desarrollo de software moderno, las APIs (Interfaces de Programación de Aplicaciones) son la columna vertebral de la comunicación entre sistemas. Postman se ha consolidado como una de las herramientas más populares y poderosas para probar, documentar y automatizar APIs. En este artículo, exploraremos cómo aplicar Postman en escenarios del mundo real, con ejemplos prácticos que puedes implementar inmediatamente.

## ¿Por qué Postman?

Postman no es solo un cliente HTTP; es una plataforma completa para el desarrollo de APIs que ofrece:

- **Interfaz intuitiva**: Ideal para principiantes y expertos
- **Automatización de pruebas**: Scripts JavaScript para validaciones complejas
- **Colecciones organizadas**: Agrupa solicitudes relacionadas
- **Variables de entorno**: Facilita el cambio entre dev, staging y producción
- **Integración CI/CD**: Compatible con Newman para pipelines automatizados
- **Colaboración en equipo**: Espacios de trabajo compartidos

## Caso de Uso 1: Prueba de API RESTful de E-commerce

Imaginemos que estamos probando una API de una tienda en línea que gestiona productos.

### Escenario: Obtener lista de productos

```javascript
// GET https://api.tienda.com/v1/productos

// En la pestaña "Tests" de Postman:
pm.test("El código de estado es 200", function () {
    pm.response.to.have.status(200);
});

pm.test("La respuesta es JSON", function () {
    pm.response.to.be.json;
});

pm.test("La respuesta contiene productos", function () {
    const jsonData = pm.response.json();
    pm.expect(jsonData).to.be.an('array');
    pm.expect(jsonData.length).to.be.above(0);
});

pm.test("Cada producto tiene campos requeridos", function () {
    const productos = pm.response.json();
    productos.forEach(producto => {
        pm.expect(producto).to.have.property("id");
        pm.expect(producto).to.have.property("nombre");
        pm.expect(producto).to.have.property("precio");
        pm.expect(producto).to.have.property("stock");
    });
});
```

### Escenario: Crear un nuevo producto (POST)

```javascript
// POST https://api.tienda.com/v1/productos
// Body (JSON):
{
    "nombre": "Laptop Dell XPS 15",
    "precio": 1299.99,
    "categoria": "Electrónica",
    "stock": 25
}

// Tests:
pm.test("Producto creado exitosamente - 201", function () {
    pm.response.to.have.status(201);
});

pm.test("La respuesta contiene el ID del producto", function () {
    const jsonData = pm.response.json();
    pm.expect(jsonData).to.have.property("id");
    
    // Guardar el ID para usar en solicitudes posteriores
    pm.environment.set("producto_id", jsonData.id);
});

pm.test("El precio se guardó correctamente", function () {
    const jsonData = pm.response.json();
    pm.expect(jsonData.precio).to.eql(1299.99);
});
```

## Caso de Uso 2: Autenticación y Autorización

Las APIs del mundo real requieren autenticación. Veamos cómo manejar tokens JWT.

### Paso 1: Login y captura de token

```javascript
// POST https://api.tienda.com/v1/auth/login
// Body:
{
    "email": "usuario@ejemplo.com",
    "password": "password123"
}

// Tests:
pm.test("Login exitoso", function () {
    pm.response.to.have.status(200);
});

pm.test("Token JWT recibido", function () {
    const jsonData = pm.response.json();
    pm.expect(jsonData).to.have.property("token");
    
    // Guardar el token en variable de entorno
    pm.environment.set("jwt_token", jsonData.token);
    
    // Guardar timestamp de expiración
    pm.environment.set("token_expiry", jsonData.expiresIn);
});
```

### Paso 2: Usar el token en solicitudes protegidas

```javascript
// GET https://api.tienda.com/v1/usuario/perfil
// Headers: Authorization: Bearer {{jwt_token}}

// Pre-request Script para verificar token:
const token = pm.environment.get("jwt_token");

if (!token) {
    throw new Error("No hay token de autenticación. Ejecuta primero el login.");
}

// Tests:
pm.test("Acceso autorizado al perfil", function () {
    pm.response.to.have.status(200);
});

pm.test("Datos del usuario son correctos", function () {
    const usuario = pm.response.json();
    pm.expect(usuario).to.have.property("email");
    pm.expect(usuario).to.have.property("nombre");
    pm.expect(usuario.rol).to.eql("admin");
});
```

## Caso de Uso 3: Encadenamiento de Solicitudes

En aplicaciones reales, las operaciones suelen ser secuenciales.

### Flujo completo: Crear pedido

```javascript
// 1. Obtener productos disponibles
// GET https://api.tienda.com/v1/productos?categoria=Electrónica

pm.test("Productos de electrónica disponibles", function () {
    const productos = pm.response.json();
    const productoDisponible = productos.find(p => p.stock > 0);
    
    // Guardar ID del primer producto disponible
    pm.environment.set("producto_seleccionado", productoDisponible.id);
    pm.environment.set("precio_producto", productoDisponible.precio);
});

// 2. Agregar al carrito
// POST https://api.tienda.com/v1/carrito
// Body:
{
    "producto_id": "{{producto_seleccionado}}",
    "cantidad": 2
}

pm.test("Producto agregado al carrito", function () {
    pm.response.to.have.status(201);
    const carrito = pm.response.json();
    pm.environment.set("carrito_id", carrito.id);
});

// 3. Crear pedido
// POST https://api.tienda.com/v1/pedidos
// Body:
{
    "carrito_id": "{{carrito_id}}",
    "direccion_envio": "Calle Principal 123",
    "metodo_pago": "tarjeta"
}

pm.test("Pedido creado exitosamente", function () {
    pm.response.to.have.status(201);
    const pedido = pm.response.json();
    
    pm.expect(pedido).to.have.property("numero_pedido");
    pm.expect(pedido.estado).to.eql("pendiente");
    
    console.log("✅ Pedido creado: " + pedido.numero_pedido);
});
```

## Caso de Uso 4: Validación de Parámetros de Búsqueda

```javascript
// GET https://api.tienda.com/v1/productos?autor=García&minPrecio=10&maxPrecio=50

pm.test("Filtrado por autor funciona correctamente", function () {
    const productos = pm.response.json();
    productos.forEach(producto => {
        pm.expect(producto.autor).to.include("García");
    });
});

pm.test("Productos dentro del rango de precio", function () {
    const productos = pm.response.json();
    productos.forEach(producto => {
        pm.expect(producto.precio).to.be.at.least(10);
        pm.expect(producto.precio).to.be.at.most(50);
    });
});

pm.test("Respuesta ordenada por relevancia", function () {
    const productos = pm.response.json();
    // Verificar que hay resultados
    pm.expect(productos.length).to.be.above(0);
});
```

## Caso de Uso 5: Pruebas de Rendimiento y Carga

### Simulación de múltiples usuarios

```javascript
// Pre-request Script para simular latencia de red
const randomDelay = Math.floor(Math.random() * 500) + 100;
setTimeout(function(){}, randomDelay);

// Tests para medir rendimiento
pm.test("Tiempo de respuesta aceptable", function () {
    pm.expect(pm.response.responseTime).to.be.below(2000); // Menos de 2 segundos
});

pm.test("No hay errores de servidor", function () {
    pm.response.to.not.have.status(500);
    pm.response.to.not.have.status(503);
});

// Guardar métricas para análisis
const tiempoRespuesta = pm.response.responseTime;
const timestamp = new Date().toISOString();

console.log(`[${timestamp}] Tiempo de respuesta: ${tiempoRespuesta}ms`);
```

## Caso de Uso 6: Manejo de Errores

```javascript
// Tests para validar manejo correcto de errores

pm.test("Error 404 cuando producto no existe", function () {
    pm.response.to.have.status(404);
});

pm.test("Mensaje de error es descriptivo", function () {
    const error = pm.response.json();
    pm.expect(error).to.have.property("mensaje");
    pm.expect(error.mensaje).to.include("no encontrado");
});

// Validar error de validación (400)
pm.test("Error 400 con datos inválidos", function () {
    pm.response.to.have.status(400);
    
    const error = pm.response.json();
    pm.expect(error).to.have.property("errores");
    pm.expect(error.errores).to.be.an('array');
});
```

## Mejores Prácticas con Postman

### 1. Organización con Colecciones

Estructura tus solicitudes de manera lógica:
```
📁 API Tienda E-commerce
  📁 Autenticación
    - POST Login
    - POST Registro
    - POST Refresh Token
  📁 Productos
    - GET Listar Productos
    - POST Crear Producto
    - PUT Actualizar Producto
    - DELETE Eliminar Producto
  📁 Pedidos
    - GET Mis Pedidos
    - POST Crear Pedido
```

### 2. Variables de Entorno

Crea entornos separados:
```
Desarrollo:
- base_url: http://localhost:3000
- api_key: dev_key_123

Staging:
- base_url: https://staging-api.tienda.com
- api_key: staging_key_456

Producción:
- base_url: https://api.tienda.com
- api_key: prod_key_789
```

### 3. Scripts Reutilizables

Usa la pestaña "Pre-request Script" a nivel de colección:
```javascript
// Función reutilizable para generar timestamps
pm.globals.set("timestamp", new Date().toISOString());

// Función para verificar token expirado
function verificarTokenExpirado() {
    const expiry = pm.environment.get("token_expiry");
    if (expiry && Date.now() > expiry) {
        console.log("⚠️ Token expirado. Renovando...");
        // Lógica para renovar token
    }
}

verificarTokenExpirado();
```

## Automatización con Newman

Para integrar tus pruebas en CI/CD:

```bash
# Instalar Newman
npm install -g newman

# Ejecutar colección
newman run mi-coleccion.json -e ambiente-produccion.json

# Con reportes HTML
newman run mi-coleccion.json -e ambiente-produccion.json -r html
```

### Ejemplo de integración con GitHub Actions

```yaml
name: API Tests
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Install Newman
        run: npm install -g newman
      - name: Run API Tests
        run: newman run postman-collection.json -e environment.json
```

## Conclusiones

Postman es mucho más que una herramienta para enviar solicitudes HTTP. Como hemos visto en estos ejemplos del mundo real:

1. **Validación robusta**: Podemos verificar códigos de estado, estructura de datos y lógica de negocio
2. **Flujos complejos**: Encadenamiento de solicitudes con variables dinámicas
3. **Automatización**: Integración perfecta con pipelines CI/CD mediante Newman
4. **Colaboración**: Espacios de trabajo compartidos facilitan el trabajo en equipo
5. **Documentación viva**: Las colecciones sirven como documentación ejecutable

### Próximos Pasos

- Explora los **monitores de Postman** para ejecutar pruebas programadas
- Implementa **mock servers** para desarrollo paralelo frontend/backend
- Aprende sobre **contract testing** con Postman
- Integra con herramientas de monitoreo como Datadog o New Relic

## Recursos Adicionales

- [Documentación oficial de Postman](https://learning.postman.com/)
- [Postman Learning Center](https://learning.postman.com/docs/getting-started/introduction/)
- [Newman en GitHub](https://github.com/postmanlabs/newman)

---

**Sobre el autor**: Este artículo forma parte del Trabajo de Investigación N° 02 sobre Comparación de Frameworks de Pruebas de API, desarrollado como parte del curso de Calidad y Pruebas de Software.

**Tags**: #API #Postman #Testing #Automatización #DevOps #QA #SoftwareTesting
