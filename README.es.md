# httpSMS - Guía de Configuración

## Tabla de Contenidos

- [Introducción](#introducción)
- [Servicios de Google Cloud Necesarios](#servicios-de-google-cloud-necesarios)
- [Configuración del Proyecto](#configuración-del-proyecto)
  - [1. Configurar Firebase](#1-configurar-firebase)
  - [2. Configurar Cloud Tasks](#2-configurar-cloud-tasks)
  - [3. Configurar Servicio SMTP](#3-configurar-servicio-smtp)
- [Control de Suscripciones](#control-de-suscripciones)
  - [Dónde se Valida la Suscripción](#dónde-se-valida-la-suscripción)
  - [Cómo Modificar la Expiración de Suscripciones](#cómo-modificar-la-expiración-de-suscripciones)
- [Configuración Local](#configuración-local)
  - [Variables de Entorno](#variables-de-entorno)
  - [Usuario del Sistema](#usuario-del-sistema)

## Introducción

httpSMS es un servicio que te permite usar tu teléfono Android como una pasarela SMS para enviar y recibir mensajes. Esta guía explica cómo configurar el ambiente necesario en Google Cloud para poner en funcionamiento la aplicación y cómo manejar el control de suscripciones para tus pruebas.

## Servicios de Google Cloud Necesarios

Para poner en funcionamiento httpSMS, necesitas habilitar los siguientes servicios en Google Cloud:

1. **Firebase**: Para autenticación, Cloud Messaging (FCM) y almacenamiento.
   - Firebase Authentication
   - Firebase Cloud Messaging
   - Firebase Storage

2. **Cloud Tasks**: Para gestionar colas de tareas asíncronas y programar eventos.

3. **(Opcional) Cloud Run**: Si deseas desplegar la API como un servicio serverless.

## Configuración del Proyecto

### 1. Configurar Firebase

1. Crea un proyecto en la [consola de Firebase](https://console.firebase.google.com/).
2. Habilita la autenticación por correo electrónico/contraseña:
   - En la sección **Authentication**, activa el método de inicio de sesión **Email/Password**.
   - Desactiva la "protección contra enumeración de correos electrónicos" ya que existe un [bug conocido](https://github.com/firebase/firebaseui-web/issues/1040).
3. Genera las credenciales de tu cuenta de servicio:
   - Ve a la configuración del proyecto > pestaña "Cuentas de servicio".
   - Haz clic en "Generar nueva clave privada" y guarda el archivo como `firebase-credentials.json`.
4. Registra tu aplicación web:
   - Sigue las [instrucciones de Firebase](https://firebase.google.com/docs/web/setup#register-app) para registrar tu aplicación web.
   // Import the functions you need from the SDKs you need
import { initializeApp } from "firebase/app";
import { getAnalytics } from "firebase/analytics";
// TODO: Add SDKs for Firebase products that you want to use
// https://firebase.google.com/docs/web/setup#available-libraries

// Your web app's Firebase configuration
// For Firebase JS SDK v7.20.0 and later, measurementId is optional
const firebaseConfig = {
  apiKey: "AIzaSyCJ49N-12JHTQTdDKfIgvfURhrIfJa2TOg",
  authDomain: "smsappltottobueno.firebaseapp.com",
  projectId: "smsappltottobueno",
  storageBucket: "smsappltottobueno.firebasestorage.app",
  messagingSenderId: "485088027957",
  appId: "1:485088027957:web:f261dfa655d58019b2187c",
  measurementId: "G-FSWMKLC76L"
};

// Initialize Firebase
const app = initializeApp(firebaseConfig);
const analytics = getAnalytics(app);
   - Guarda la configuración del SDK web (apiKey, authDomain, etc.).
5. Registra tu aplicación Android:
   - Sigue las [instrucciones](https://support.google.com/firebase/answer/7015592?hl=es#android) para obtener el archivo `google-services.json`.

   Para que los SDK de Firebase puedan acceder a los valores de configuración de google-services.json, necesitas el complemento Gradle de los servicios de Google.


DSL de Kotlin (build.gradle.kts)

Groovy (build.gradle)
Agrega el complemento como una dependencia a tu archivo build.gradle.kts de nivel de proyecto:

Archivo de Gradle de nivel de raíz (nivel de proyecto) (<project>/build.gradle.kts):
plugins {
  // ...

  // Add the dependency for the Google services Gradle plugin
  id("com.google.gms.google-services") version "4.4.2" apply false

}
Luego, en el archivo build.gradle.kts del módulo (nivel de la app), agrega el complemento google-services y cualquier SDK de Firebase que quieras usar en tu app:

Archivo de Gradle del módulo (nivel de app) (<project>/<app-module>/build.gradle.kts):
plugins {
  id("com.android.application")

  // Add the Google services Gradle plugin
  id("com.google.gms.google-services")

  ...
}

dependencies {
  // Import the Firebase BoM
  implementation(platform("com.google.firebase:firebase-bom:33.12.0"))


  // TODO: Add the dependencies for Firebase products you want to use
  // When using the BoM, don't specify versions in Firebase dependencies
  implementation("com.google.firebase:firebase-analytics")


  // Add the dependencies for any other desired Firebase products
  // https://firebase.google.com/docs/android/setup#available-libraries
}

### 2. Configurar Cloud Tasks

1. Activa la API Cloud Tasks en la [consola de Google Cloud](https://console.cloud.google.com/):
   - Busca "Cloud Tasks API" en el buscador y actívala.
2. Configura una cola de tareas:
   - Esta cola se utilizará para gestionar eventos asincrónicos como el envío de mensajes.

### 3. Configurar Servicio SMTP

La aplicación utiliza SMTP para enviar correos electrónicos a los usuarios. Puedes usar servicios como:
- [Mailtrap](https://mailtrap.io/) para desarrollo
- [SendGrid](https://sendgrid.com/)
- [Mailgun](https://www.mailgun.com/)

Necesitarás las credenciales SMTP (usuario, contraseña, host, puerto) para la configuración.

## Control de Suscripciones

### Dónde se Valida la Suscripción

El sistema de suscripciones de httpSMS se gestiona principalmente a través de los siguientes archivos:

1. **`api/pkg/entities/user.go`**: Define los tipos de suscripción y sus límites.
2. **`api/pkg/services/billing_service.go`**: Gestiona la validación de derechos y límites de usuario.
3. **`api/pkg/entities/billing_usage.go`**: Contiene la lógica para verificar si un usuario puede enviar mensajes.

### Cómo Modificar la Expiración de Suscripciones

Para tu entorno de pruebas, puedes modificar el comportamiento de las suscripciones de varias formas:

1. **Modificación en la base de datos**:
   - Puedes actualizar directamente los campos de suscripción del usuario en la tabla `users`:
   ```sql
   UPDATE users
   SET subscription_name = 'pro-lifetime',
       subscription_ends_at = NULL,
       subscription_renews_at = NULL
   WHERE id = 'tu-id-usuario';
   ```

2. **Modificar el método `IsEntitled` en `billing_service.go`**:
   - Puedes modificar temporalmente el método `IsEntitled` para que siempre retorne `nil` (lo que significa que el usuario está autorizado):
   ```go
   func (service *BillingService) IsEntitled(ctx context.Context, userID entities.UserID) *string {
       // Para pruebas, siempre autorizar
       return nil
   }
   ```

3. **Crear un usuario con suscripción de por vida**:
   - Crea un usuario y asígnale la suscripción `SubscriptionNameProLifetime` que no tiene fecha de expiración.

4. **Interceptar eventos de expiración**:
   - Modifica el método `ExpireSubscription` en `user_service.go` para que no cambie la suscripción del usuario cuando reciba un evento de expiración.

## Configuración Local

### Variables de Entorno

1. Copia los archivos de ejemplo y actualízalos:
   ```bash
   cp web/.env.docker web/.env.local
   cp api/.env.docker api/.env.local
   ```

2. Actualiza las variables de entorno con tus credenciales de Firebase y SMTP.

### Usuario del Sistema

El sistema requiere un usuario especial para procesar eventos asíncronos:

1. Crea un usuario "del sistema" manualmente en la tabla `users` de tu base de datos.
2. Asegúrate de usar el mismo `id` y `api_key` que configuraste en `EVENTS_QUEUE_USER_ID` y `EVENTS_QUEUE_USER_API_KEY` en tu archivo `.env`.

Con estas configuraciones, deberías poder ejecutar httpSMS en tu entorno local o de pruebas con control total sobre las suscripciones. 