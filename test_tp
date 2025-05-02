#include <pthread.h>     
#include <semaphore.h>   
#include <stdio.h>      
#include <stdlib.h>      
#include <time.h>        
#include <unistd.h>    


#define N_BUS_X 5                   
#define N_BUS_Y 4                  
#define MAX_TRAJETS 10              
#define MAX_CONSECUTIVE_BUSES 3     // max de bus consecutif dans un même sens dans le tunnel

// declaration des sémaphores 
sem_t mutex;         
sem_t attente_X;      
sem_t attente_Y;      

// variable partagé
int nb_bus_dans_tunnel = 0;         
char sens_courant = 'X';            
int en_attente_X = 0;               
int en_attente_Y = 0;               
int compteur_consecutive = 0;       


void attendre_duree_random() { 
    usleep((rand() % 500 + 1000) * 1000); 
}

// fonction pour faire entrer un bus dans le tunnel 
void entrer_tunnel(char sens, int id) {
  sem_wait(&mutex);  // Entrée dans section critique

  if (nb_bus_dans_tunnel == 0) {
    sens_courant = sens;
    compteur_consecutive = 0;
  }

  if (nb_bus_dans_tunnel == 0 && compteur_consecutive >= MAX_CONSECUTIVE_BUSES) {
    if (sens == 'X' && en_attente_Y > 0) {
      sens_courant = 'Y';
      compteur_consecutive = 0;
      printf("Changement forcé pour équité: désormais Y->X\n");
    } else if (sens == 'Y' && en_attente_X > 0) {
      sens_courant = 'X';
      compteur_consecutive = 0;
      printf("Changement forcé pour équité: désormais X->Y\n");
    }
  }

  if (sens == sens_courant) {
    nb_bus_dans_tunnel++;
    compteur_consecutive++;
    sem_post(&mutex);  // Sortie de section critique
  } else {
   
    if (sens == 'X') {
      en_attente_X++;
      sem_post(&mutex);      
      sem_wait(&attente_X);  

      sem_wait(&mutex);
      nb_bus_dans_tunnel++;
      compteur_consecutive++;
      sem_post(&mutex);
    } else {
      en_attente_Y++;
      sem_post(&mutex);
      sem_wait(&attente_Y);

      sem_wait(&mutex);
      nb_bus_dans_tunnel++;
      compteur_consecutive++;
      sem_post(&mutex);
    }
  }
}

// fonction pour faire sortir un bus du tunnel 
void sortir_tunnel(char sens) {
  sem_wait(&mutex);
  nb_bus_dans_tunnel--;

  // Si plus aucun bus dans le tunnel
  if (nb_bus_dans_tunnel == 0) {
    if ((compteur_consecutive >= MAX_CONSECUTIVE_BUSES)  
     || (sens == 'X' && en_attente_Y > 0)  
     || (sens == 'Y' && en_attente_X > 0)) 
    {
      // Inversion du sens si nécessaire
      if (sens == 'X' && en_attente_Y > 0) {
        sens_courant = 'Y';
        compteur_consecutive = 0;
        printf("Changement de sens: désormais Y->X (%d bus en attente)\n", en_attente_Y);

        int nb_a_reveiller = (en_attente_Y > MAX_CONSECUTIVE_BUSES) ? 
                              MAX_CONSECUTIVE_BUSES : en_attente_Y;

        for (int i = 0; i < nb_a_reveiller; i++) {
          sem_post(&attente_Y);
        }
        en_attente_Y -= nb_a_reveiller;

      } else if (sens == 'Y' && en_attente_X > 0) {
        sens_courant = 'X';
        compteur_consecutive = 0;
        printf("Changement de sens: désormais X->Y (%d bus en attente)\n", en_attente_X);

        int nb_a_reveiller = (en_attente_X > MAX_CONSECUTIVE_BUSES) ? 
                              MAX_CONSECUTIVE_BUSES : en_attente_X;

        for (int i = 0; i < nb_a_reveiller; i++) {
          sem_post(&attente_X);
        }
        en_attente_X -= nb_a_reveiller;

      } else {
        compteur_consecutive = 0;
      }
    }
  }
  sem_post(&mutex);
}

typedef struct {
  int id;        
  char depart;   
} BusArgs;

// fonction exécutée par chaque thread de bus
void *bus_fonction(void *arg) {
  BusArgs *bus = (BusArgs *)arg;
  char ville_depart = bus->depart;
  int id = bus->id;
  char ville_origine = ville_depart;

  for (int i = 1; i <= MAX_TRAJETS; i++) {
    
    char ville_arrivee = (ville_depart == 'X') ? 'Y' : 'X';
    entrer_tunnel(ville_depart, id);
    printf("Bus [%d] de [Ville %c] : [Ville %c] -> "
           "[Ville %c] (Trajet [%d])\n",
           id, ville_origine, ville_depart,
           ville_arrivee, i);
    attendre_duree_random();  
    sortir_tunnel(ville_depart);

    ville_depart = ville_arrivee;

   
    ville_arrivee = (ville_depart == 'X') ? 'Y' : 'X';
    entrer_tunnel(ville_depart, id);
    printf("Bus [%d] de [Ville %c] : [Ville %c] -> "
           "[Ville %c] (Trajet [%d])\n",
           id, ville_origine, ville_depart,
           ville_arrivee, i);
    attendre_duree_random();
    sortir_tunnel(ville_depart);

    ville_depart = ville_arrivee;  
  }

  free(bus);
  pthread_exit(NULL);
}


int main() {
  srand(time(NULL));  

  sem_init(&mutex, 0, 1);
  sem_init(&attente_X, 0, 0);
  sem_init(&attente_Y, 0, 0);

  pthread_t threads[N_BUS_X + N_BUS_Y];

  // creation des threads de bus de la ville X
  for (int i = 0; i < N_BUS_X; i++) {
    BusArgs *args = malloc(sizeof(BusArgs));
    args->id = i + 1;
    args->depart = 'X';
    pthread_create(&threads[i], NULL, bus_fonction, args);
  }

  //creation des threads de bus de la ville Y
  for (int i = 0; i < N_BUS_Y; i++) {
    BusArgs *args = malloc(sizeof(BusArgs));
    args->id = i + N_BUS_X + 1;
    args->depart = 'Y';
    pthread_create(&threads[N_BUS_X + i], NULL, bus_fonction, args);
  }

  //attente de la fin de tous les threads
  for (int i = 0; i < N_BUS_X + N_BUS_Y; i++) {
    pthread_join(threads[i], NULL);
  }

  //destruction des sémaphores
  sem_destroy(&mutex);
  sem_destroy(&attente_X);
  sem_destroy(&attente_Y);
  return 0;
}
