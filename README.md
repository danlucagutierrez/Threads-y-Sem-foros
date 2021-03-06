# MiniTP Semáforos y Threads

- Docentes: Ignacio Tula, Mariano Vargas
- Alumno: Dan Luca Gutierrez
- Comisión: 01


## Link del repositorio
https://github.com/danlucagutierrez/Threads-y-Sem-foros

![](https://i.pinimg.com/originals/1c/61/d4/1c61d40751eb1efdac991bb129b8c970.jpg)


### Introducción
En este trabajo práctico se implemento en lenguaje C, mediante semáforos y threads la lógica para simular una competencia entre equipos a realizar el mejor sanguche de milanesa.

=============

#### Comando para ejecutar
`$ ./subwayArgento_exe`

#### Código fuente
```python
#include <stdio.h>	//Libreria estandar
#include <stdlib.h> 	//Para usar exit y funciones de la libreria standard
#include <string.h>
#include <pthread.h>    //Para usar threads
#include <semaphore.h>  //Para usar semaforos
#include <unistd.h>

//Crear archivo de resultado
        char *nombre_archivo = "resultado.txt";
        char *modo = "a+";

//Creo semaforos Mutex para acceder a sección crítica entre hilos (entre equipos)
	sem_t sem_horno_libre;
	sem_t sem_salero_libre;
	sem_t sem_sarten_libre;
	sem_t sem_ganar;

//Variable necesaria
	char *estado_termino = "terminar";
	int ban_ganar = 1;

#define LIMITE 50

//Creo estructura de semaforos para acceder a sección crítica interna de cada hilo (entre compañeros de cada equipo)
struct semaforos{
	sem_t sem_mezclar;
	sem_t sem_salar;
	sem_t sem_empanar;
	sem_t sem_cocinar;
	sem_t sem_cortar_otros;
	sem_t sem_hornear;
	sem_t sem_armar;
	sem_t sem_terminar;
};

//Creo los pasos con los ingredientes
struct paso{
	char accion[LIMITE];
	char ingredientes[4][LIMITE];
};

//Creo los parametros de los hilos
struct parametro{
	int equipo_param;
	struct semaforos semaforos_param;
	struct paso pasos_param[8];
};

//Funcion para imprimir las acciones y los ingredientes de la accion
void *imprimirAccion(void *data, char *accionIn){
	struct parametro *mydata = data;
	FILE *archivo = fopen(nombre_archivo, modo);
	//Checkea si termino cada hilo
	if(strcmp(estado_termino, accionIn) == 0){
		printf("\tEquipo %d - termino! \n ", mydata->equipo_param);
		fprintf(archivo, "\tEquipo %d - termino! \n ", mydata->equipo_param);
		//Checkea cual de los hilos termino primero
		sem_wait(&sem_ganar);
		if(ban_ganar == 1){
			ban_ganar = 0;
			printf("\tEquipo %d - gano! \n ", mydata->equipo_param);
			fprintf(archivo, "\tEquipo %d - gano! \n ", mydata->equipo_param);
		}
	}
	//Calculo la longitud del array de pas
	int sizeArray = (int)(sizeof(mydata->pasos_param) / sizeof(mydata->pasos_param[0]));
	//Indice para recorrer array de pasos
	int i;
	for(i = 0; i < sizeArray; i++){
		//Pregunto si la accion del array es igual a la pasada por parametro (si es igual la funcion strcmp devuelve cero)
		if(strcmp(mydata->pasos_param[i].accion, accionIn) == 0){
			printf("\tEquipo %d - accion %s \n ", mydata->equipo_param, mydata->pasos_param[i].accion);
		fprintf(archivo, "\tEquipo %d - accion %s \n ", mydata->equipo_param, mydata->pasos_param[i].accion);


			//Calculo la longitud del array de ingredientes
			int sizeArrayIngredientes = (int)(sizeof(mydata->pasos_param[i].ingredientes) / sizeof(mydata->pasos_param[i].ingredientes[0]));
			//Indice para recorrer array de ingredientes
			int h;
			printf("\tEquipo %d ----------- ingredientes : ----------\n", mydata->equipo_param);
        	fprintf(archivo,"\tEquipo %d ----------- ingredientes : ----------\n", mydata->equipo_param);
			for(h = 0; h < sizeArrayIngredientes; h++){
				//Consulto si la posicion tiene valor porque no se cuantos ingredientes tengo por accion
				if(strlen(mydata->pasos_param[i].ingredientes[h]) != 0){
					printf("\tEquipo %d ingrediente  %d : %s \n", mydata->equipo_param, h, mydata->pasos_param[i].ingredientes[h]);
					fprintf(archivo, "\tEquipo %d ingrediente  %d : %s \n", mydata->equipo_param,h,mydata->pasos_param[i].ingredientes[h]);
				}
			}
		}
	}
}

//Funcion para tomar de ejemplo
void *cortar(void *data){
	//Creo el nombre de la accion de la funcion
	char *accion = "cortar";
	//Creo el puntero para pasarle la referencia de memoria (data) del struct pasado por parametro (la cual es un puntero).
	struct parametro *mydata = data;
	//Llamo a la funcion imprimir le paso el struct y la accion de la funcion
	imprimirAccion(mydata, accion);
	//Uso sleep para simular que que pasa tiempo
	usleep(2000000);
	//Doy la señal a la siguiente accion (cortar me habilita mezclar)
	sem_post(&mydata->semaforos_param.sem_mezclar);
	pthread_exit(NULL);
}

void *cortarOtros(void *data){
	char *accion = "cortar otros";
        struct parametro *mydata = data;
	imprimirAccion(mydata, accion);
        usleep(2000000);
	sem_post(&mydata->semaforos_param.sem_cortar_otros);
        pthread_exit(NULL);
}

void *mezclar(void *data){
	char *accion = "mezclar";
	struct parametro *mydata = data;
	sem_wait(&mydata->semaforos_param.sem_mezclar);
	imprimirAccion(mydata, accion);
	usleep(3000000);
	sem_post(&mydata->semaforos_param.sem_salar);
	pthread_exit(NULL);
}

void *salar(void *data){
	char *accion = "salar";
	struct parametro *mydata = data;
	sem_wait(&mydata->semaforos_param.sem_salar);
	sem_wait(&sem_salero_libre);
	imprimirAccion(mydata, accion);
	usleep(1000000);
	sem_post(&mydata->semaforos_param.sem_empanar);
	sem_post(&sem_salero_libre);
	pthread_exit(NULL);
}

void *empanar(void *data){
	char *accion = "empanar";
	struct parametro *mydata = data;
	sem_wait(&mydata->semaforos_param.sem_empanar);
	imprimirAccion(mydata, accion);
	usleep(1000000);
	sem_post(&mydata->semaforos_param.sem_cocinar);
	pthread_exit(NULL);
}

void *cocinar(void *data){
	char *accion = "cocinar";
	struct parametro *mydata = data;
	sem_wait(&mydata->semaforos_param.sem_cocinar);
	sem_wait(&sem_sarten_libre);
	imprimirAccion(mydata, accion);
	usleep(5000000);
	sem_post(&mydata->semaforos_param.sem_hornear);
	sem_post(&sem_sarten_libre);
	pthread_exit(NULL);
}

void *hornear(void *data){
	char *accion = "hornear";
	struct parametro *mydata = data;
	sem_wait(&mydata->semaforos_param.sem_hornear);
	sem_wait(&sem_horno_libre);
	imprimirAccion(mydata, accion);
	usleep(10000000);
	sem_post(&sem_horno_libre);
	sem_post(&mydata->semaforos_param.sem_armar);
	pthread_exit(NULL);
}

void *armar(void *data){
	char *accion = "armar";
	struct parametro *mydata = data;
	sem_wait(&mydata->semaforos_param.sem_cortar_otros);
	sem_wait(&mydata->semaforos_param.sem_armar);
	imprimirAccion(mydata, accion);
	usleep(1000000);
	sem_post(&mydata->semaforos_param.sem_terminar);
	pthread_exit(NULL);
}

void *terminar(void *data){
	char *accion = "terminar";
	struct parametro *mydata = data;
	sem_wait(&mydata->semaforos_param.sem_terminar);
	sem_post(&sem_ganar);
	imprimirAccion(mydata, accion);
	pthread_exit(NULL);
}

void *ejecutarReceta(void *i){
	//Variables semaforos internos
	sem_t sem_mezclar;
	sem_t sem_salar;
	sem_t sem_empanar;
	sem_t sem_cocinar;
	sem_t sem_hornear;
	sem_t sem_cortar_otros;
	sem_t sem_armar;
	sem_t sem_terminar;

	//Variables hilos internos
	pthread_t p1;
	pthread_t p2;
	pthread_t p3;
	pthread_t p4;
	pthread_t p5;
	pthread_t p6;
	pthread_t p7;
	pthread_t p8;
	pthread_t p9;

	//Numero del equipo (casteo el puntero a un int)
	int p = *((int *)i);

	printf("Ejecutando equipo %d \n", p);

	//Reservo memoria para el struct
	struct parametro *pthread_data = malloc(sizeof(struct parametro));

	//Seteo numero de grupo
	pthread_data->equipo_param = p;

	//Seteo semaforos
	pthread_data->semaforos_param.sem_mezclar = sem_mezclar;
	pthread_data->semaforos_param.sem_salar = sem_salar;
	pthread_data->semaforos_param.sem_empanar = sem_empanar;
	pthread_data->semaforos_param.sem_cocinar = sem_cocinar;
	pthread_data->semaforos_param.sem_hornear = sem_hornear;
	pthread_data->semaforos_param.sem_cortar_otros = sem_cortar_otros;
	pthread_data->semaforos_param.sem_armar = sem_armar;
        pthread_data->semaforos_param.sem_terminar = sem_terminar;

	//Seteo las acciones y los ingredientes (Faltan acciones e ingredientes) ¿Se ve hardcodeado no? ¿Les parece bien?
	strcpy(pthread_data->pasos_param[0].accion, "cortar");
        strcpy(pthread_data->pasos_param[0].ingredientes[0], "ajo");
        strcpy(pthread_data->pasos_param[0].ingredientes[1], "perejil");

        strcpy(pthread_data->pasos_param[1].accion, "mezclar");
        strcpy(pthread_data->pasos_param[1].ingredientes[0], "ajo");
        strcpy(pthread_data->pasos_param[1].ingredientes[1], "perejil");
        strcpy(pthread_data->pasos_param[1].ingredientes[2], "huevo");
        strcpy(pthread_data->pasos_param[1].ingredientes[3], "carne");

        strcpy(pthread_data->pasos_param[2].accion, "salar");
        strcpy(pthread_data->pasos_param[2].ingredientes[0], "sal");
	strcpy(pthread_data->pasos_param[2].ingredientes[1], "ingredientes");

        strcpy(pthread_data->pasos_param[3].accion, "empanar");
        strcpy(pthread_data->pasos_param[3].ingredientes[0], "pan rallado");
        strcpy(pthread_data->pasos_param[3].ingredientes[1], "carne");
        strcpy(pthread_data->pasos_param[3].ingredientes[2], "ingredientes");

        strcpy(pthread_data->pasos_param[4].accion, "cocinar");
        strcpy(pthread_data->pasos_param[4].ingredientes[0], "milanesa");

        strcpy(pthread_data->pasos_param[5].accion, "hornear");
        strcpy(pthread_data->pasos_param[5].ingredientes[0], "panes");

        strcpy(pthread_data->pasos_param[6].accion, "cortar otros");
        strcpy(pthread_data->pasos_param[6].ingredientes[0], "lechuga");
        strcpy(pthread_data->pasos_param[6].ingredientes[1], "tomate");
        strcpy(pthread_data->pasos_param[6].ingredientes[2], "cebolla morada");
        strcpy(pthread_data->pasos_param[6].ingredientes[3], "pepino");

        strcpy(pthread_data->pasos_param[7].accion, "armar");
        strcpy(pthread_data->pasos_param[7].ingredientes[0], "milanesa");
        strcpy(pthread_data->pasos_param[7].ingredientes[1], "pan");
        strcpy(pthread_data->pasos_param[7].ingredientes[2], "ingredientes");

	//Inicializo los semaforos
	sem_init(&(pthread_data->semaforos_param.sem_mezclar), 0, 0);
	sem_init(&(pthread_data->semaforos_param.sem_salar), 0, 0);
	sem_init(&(pthread_data->semaforos_param.sem_empanar), 0, 0);
	sem_init(&(pthread_data->semaforos_param.sem_cocinar), 0, 0);
	sem_init(&(pthread_data->semaforos_param.sem_cortar_otros), 0, 0);
	sem_init(&(pthread_data->semaforos_param.sem_hornear), 0, 0);
	sem_init(&(pthread_data->semaforos_param.sem_armar), 0, 0);
        sem_init(&(pthread_data->semaforos_param.sem_terminar), 0, 0);

	//Creo los hilos internos a todos les paso el struct creado (el mismo a todos los hilos) ya que todos comparten los semaforos
	int rc;
	rc = pthread_create(&p1, NULL, cortar, pthread_data);
	rc = pthread_create(&p2, NULL, mezclar, pthread_data);
	rc = pthread_create(&p3, NULL, salar, pthread_data);
	rc = pthread_create(&p4, NULL, empanar, pthread_data);
	rc = pthread_create(&p5, NULL, cocinar, pthread_data);
	rc = pthread_create(&p7, NULL, cortarOtros, pthread_data);
	rc = pthread_create(&p6, NULL, hornear, pthread_data);
	rc = pthread_create(&p8, NULL, armar, pthread_data);
	rc = pthread_create(&p9, NULL, terminar, pthread_data);

	//Join de todos los hilos
	pthread_join(p1, NULL);
	pthread_join(p2, NULL);
	pthread_join(p3, NULL);
	pthread_join(p4, NULL);
	pthread_join(p5, NULL);
	pthread_join(p6, NULL);
	pthread_join(p7, NULL);
	pthread_join(p8, NULL);
       // pthread_join(p9, NULL);

	//Valido que el hilo se alla creado bien
	if (rc){
		printf("Error:unable to create thread, %d \n", rc);
		exit(-1);
	}

	//Destruccion de los semaforos internos
	sem_destroy(&sem_mezclar);
	sem_destroy(&sem_salar);
	sem_destroy(&sem_empanar);
	sem_destroy(&sem_cocinar);
	sem_destroy(&sem_cortar_otros);
	sem_destroy(&sem_hornear);
	sem_destroy(&sem_armar);
	sem_destroy(&sem_terminar);

	//Salida del hilo
	pthread_exit(NULL);
}

int main(){
	//Creo los nombres de los equipos
	int rc;
	int *equipoNombre1 = malloc(sizeof(*equipoNombre1));
	int *equipoNombre2 = malloc(sizeof(*equipoNombre2));
	int *equipoNombre3 = malloc(sizeof(*equipoNombre3));
	int *equipoNombre4 = malloc(sizeof(*equipoNombre4));

	*equipoNombre1 = 1;
	*equipoNombre2 = 2;
	*equipoNombre3 = 3;
	*equipoNombre4 = 4;

	//Creo las variables los hilos de los equipos
	pthread_t equipo1;
	pthread_t equipo2;
	pthread_t equipo3;
	pthread_t equipo4;

	//Inicializo los semaforos externos
	sem_init((&sem_horno_libre), 0, 1);
	sem_init((&sem_salero_libre), 0, 1);
	sem_init((&sem_sarten_libre), 0, 1);
	sem_init((&sem_ganar), 0, 0);

	//Inicializo los hilos de los equipos
	rc = pthread_create(&equipo1, NULL, ejecutarReceta, equipoNombre1);
	rc = pthread_create(&equipo2, NULL, ejecutarReceta, equipoNombre2);
	rc = pthread_create(&equipo3, NULL, ejecutarReceta, equipoNombre3);
	rc = pthread_create(&equipo4, NULL, ejecutarReceta, equipoNombre4);

	if (rc){
		printf("Error:unable to create thread, %d \n", rc);
		exit(-1);
	}

	//Join de todos los hilos
	pthread_join(equipo1, NULL);
	pthread_join(equipo2, NULL);
	pthread_join(equipo3, NULL);
	pthread_join(equipo4, NULL);

	//Cierro los semaforos externos
	sem_destroy(&sem_horno_libre);
	sem_destroy(&sem_salero_libre);
	sem_destroy(&sem_sarten_libre);
	sem_destroy(&sem_ganar);

	//Cerramos los hilos
	pthread_exit(NULL);
}
```
----

### Funciones importantes para los hilos y semáforos

| Function name | Description                    |
| ------------- | ------------------------------ |
| `main()`      | Se crean los hilos que represetan a los 4 equipos y los semáforos externos. |
| `ejecutarReceta()`   | Se crean los hilos que corren dentro de cada equipo, y los semaforos internos. |
| `imprimirAccion()`      | Recorre la receta y muestra la acción y los ingredientes, además escribe en un archivo llamado resultados.txt |

## Dificultades y decisiones
Para leer y entender el template de base tuve que investigar sobre C, cuestiones de punteros, direcciones de memoria, strucs y las expresiones (*, &, ->).

Para implementar semáforos y threads busque en la documentación y la información que se nos fue dando en las clases, ya que no fue facil entender el funcionamiento de los mismos.

En primera instancia aclarar que para el archivo resultados.txt (no pude encontrar la forma de cerrarlo una vez que se termina de escribir con fclose(), y tampoco resolví el problema de que se escriba primero lo último y en lo último lo primero).

Por otro lado tome decisiones durante la implementación que deberian ser aclaradas:
- Los equipos inicializan en paralelo, cada uno en su hilo.
- Cada equipo o hilo realiza acciones de forma concurrente (en hilos diferentes) pero solo las acciónes cortar() y cortarOtros() se puede realizar de forma paralela, para las demás se necesita preguntar a los semáforos antes de acceder a la zona crítica.
- Dos o más equipos no pueden usar la sal, la sarten o la cocina al mismo tiempo (de forma paralela) por lo tanto se utilizaron semaforos 'externos' que habilitan o no el uso de estas acciones.
- No se lee desde un archivo la receta, si no más bien esta harcodeada en un arreglo.
- Se resuelve la receta siguiendo el orden de las posiciones del arreglo.
- Todos los equipos lanzan el mensaje de "Equipo % termino.", solo el equipo que termino primero lanza el mensaje "Equipo % gano.".
- Se resuelve esto último sobre el mensaje gano, utilizando un semaforo externo para todos los hilos y una bandera (una variable binaria que toma dos valores 0 o 1, ya que en C no existen las variables booleanas).

## Conclusiónes
El resultado de hacer el tp fue principalmente aclarar dudas y cuestiones que se presentan cuando uno lee la teoria y resuelve ejercicios con pseudocodigo. Fue importante observar lo que sucedía con *htop* ya que en varias oportunidades la ejecución no terminaba y se podía observar como quedaban hilos en proceso que ~~no se mataban~~. 
¡Muy interesante este MiniTP!
