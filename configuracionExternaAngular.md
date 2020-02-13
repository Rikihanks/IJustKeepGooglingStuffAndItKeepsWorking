


# Angular

## Cómo obtener variables de configuración desde un fichero externo en angular.

En angular se ha extendido el uso de los **env (environments)** para la carga de variables que pueden cambiar en base a en qué entorno nos encontremos. 

Personalmente no creo que esta sea la mejor forma de hacerlo puesto que nos obliga a compilar el proyecto cada vez que queremos cambiar dichos archivos.

En este artículo aprenderemos como cargar variables de configuración desde un archivo externo, ahorrándonos tener que compilar el proyecto para nuestros diferentes entornos.

Doy por hecho que ya tenéis un archivo .json con los valores a cargar. En mi caso usaré este:

    {
    	"baseUrl": "http://miUrl.org/api/v1"
    }

### 1. Crear el servicio.
Primero crearemos el servicio que se encargará de inyectar las variables con los datos que vengan de nuestro archivo de configuración. 

Como siempre en angular `ng g s nombreDelServicio`
Una vez creado procedemos a configurarlo:
Primero tenemos que inyectar el servicio HttpClient en el constructor para poder usarlo.

    constructor(private  http:  HttpClient) { }

Una vez lo tenemos definiremos las variables de configuración que vamos a usar, por simplificar en este caso solo usaré la variable "baseUrl".

    constructor(private  http:  HttpClient) { }
    baseUrl:  string;
  
Una vez definidas, tenemos que crear un método que será el que se encargue de asegurarse de que nuestras variables han tomado su valor desde el archivo de configuración, yo lo llamaré "ensureInit". Este método devolverá una `Promise<any>`   

    ensureInit():Promise<any> {
    }
    
*Ok, ¿y qué hago con esto?*

Ahora vamos a definir el contenido del método "ensureInit".

Lo más probable es que vuestro archivo de configuración esté en vuestro proyecto, en la carpeta assets por ejemplo.
En ese caso devolveríamos una promesa donde hacemos una petición http a nuestro propio proyecto para que nos devuelva el archivo, y gracias a un método maravilloso de Javascript, `(Object.assing())` copiamos todos sus valores a nuestra clase.
**Nota =>** Los nombres de las variables del archivo y los nombres de las variables del servicio deben ser iguales.**<= Nota**

El método quedaría así:

      ensureInit():Promise<any> {
       return new Promise((r, e) => {
            this.http.get("assets/config/config.json")
              .subscribe(
                (content: AppConfigService) => {
                  Object.assign(this, content);
                  r(this);
                },
                reason => e(reason));
          })
       }
       
Donde: La url sería el directorio donde tengáis el archivo config.json
¿Habéis visto la magia de `Object.assign()`?

*¿Y si quiero que el archivo esté completamente fuera del proyecto?*

Pues sería exactamente lo mismo pero cambiando la url actual por aquella desde donde estés sirviendo tu archivo de configuración. Por ejemplo:

    ensureInit():Promise<any> {
           return new Promise((r, e) => {
                this.http.get("https://miconfiguracion.com/config.json")
                  .subscribe(
                    (content: AppConfigService) => {
                      Object.assign(this, content);
                      r(this);
                    },
                    reason => e(reason));
              })
           }
           
Ahora que ya tenemos el servicio es hora de movernos a la parte de decirle a Angular que no arranque hasta que éste no se haya cargado.

### 2. Hacer que angular dependa del servicio
Para esto tenemos que añadir unas líneas en el `app.module.ts`

Seguro que el vuestro se parece un poco a este:

    import { NgModule, APP_INITIALIZER } from '@angular/core';
    import { BrowserModule } from '@angular/platform-browser';
    import { FormsModule } from '@angular/forms';
    
    import { AppComponent } from './app.component';
    import { HelloComponent } from './hello.component';
    import {HttpClientModule} from '@angular/common/http';
    import { HttpClient } from '@angular/common/http';
    import { AppConfiguration } from './app-configuration.service';
    @NgModule({
      imports:      [ BrowserModule, FormsModule, HttpClientModule ],
      declarations: [ AppComponent, HelloComponent ],
       providers: [],
      bootstrap:    [ AppComponent ]
    })
    export class AppModule { }
    
Lo que haremos será añadir un nuevo provider. En nuestro caso, el del servicio que acabamos de crear, y añadirle un par de configuraciones que explicaré ahora.

    import { NgModule, APP_INITIALIZER } from '@angular/core';
    import { BrowserModule } from '@angular/platform-browser';
    import { FormsModule } from '@angular/forms';
    
    import { AppComponent } from './app.component';
    import { HelloComponent } from './hello.component';
    import {HttpClientModule} from '@angular/common/http';
    import { HttpClient } from '@angular/common/http';
    import { AppConfigurationService } from './app-configuration.service';
    @NgModule({
      imports:      [ BrowserModule, FormsModule, HttpClientModule ],
      declarations: [ AppComponent, HelloComponent ],
       providers: [
         AppConfigurationService,
        { 
            provide: APP_INITIALIZER, 
            useFactory: AppConfigurationFactory, 
            deps: [AppConfiguration, HttpClient], multi: true },
      ],
      bootstrap:    [ AppComponent ]
    })
    export class AppModule { }
    
 **APP_INITIALIZER** le dice a angular que tiene que esperar a que este provider se inicie antes de arrancar los demás.
 
 **UseFactory** recibirá como parámetro una función que lo que hará será llamar al método ensureInit que creamos antes para asegurarse de que nuestra clase de configuración tiene sus valores seteados.
 
 **deps:** Son las dependencias del provider, en este caso el servicio en sí mismo y el cliente http que usamos dentro del servicio.
 
*¿Y dónde creo la función que llama al ensureInit?*

Pues en el propio `app.module.ts`. Justo después de `export class appModule{}` definiremos nuestra función **AppConfigurationFactory**

    export function AppConfigurationFactory(
      appConfig: AppConfigService) {
      return () => appConfig.ensureInit();
    }
    
Como veis lo único que hace es llamar al método que creamos antes en el servicio.
El app.module completo quedaría así:

    import { NgModule, APP_INITIALIZER } from '@angular/core';
    import { BrowserModule } from '@angular/platform-browser';
    import { FormsModule } from '@angular/forms';
    
    import { AppComponent } from './app.component';
    import { HelloComponent } from './hello.component';
    import {HttpClientModule} from '@angular/common/http';
    import { HttpClient } from '@angular/common/http';
    import { AppConfiguration } from './app-configuration.service';
    @NgModule({
      imports:      [ BrowserModule, FormsModule, HttpClientModule ],
      declarations: [ AppComponent, HelloComponent ],
       providers: [
         AppConfiguration,
        { 
            provide: APP_INITIALIZER, 
            useFactory: AppConfigurationFactory, 
            deps: [AppConfiguration, HttpClient], multi: true },
      ],
      bootstrap:    [ AppComponent ]
    })
    export class AppModule { }
    
    export function AppConfigurationFactory(
    appConfig: AppConfiguration) {
      return () => appConfig.ensureInit();
    }

Si todo ha ido bien, ahora podríais inyectar el servicio como cualquier otro  en un componente de angular y obtener los valores que provienen de nuestro archivo json
