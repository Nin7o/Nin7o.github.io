---
title: "Minishell"
permalink: /minishell/
author_profile: false
redirect_from:
  - /minishell
---

Download an archive of the project : [Click here](/files/minishell.zip)

Dynamic allocated list for job storage implementation : 
==

```c
#include "job.h"

// Nombre de processus en cours
int nbjobs = 0;

// compteur pour définir les id des processus
int job_id_counter = 0;

// Liste des processus
job * jobs = NULL;

/*
 * ajoute un processus à la liste des processus
*/
void add_job(int id, pid_t pid, char * state, char** cmd) {

	job *j = malloc(sizeof(job));
	j->id = id;
	j->pid = pid;
	j->state = strdup(state);
	
	int nbargscmd = 0;

	// compter le nombre d'arguments
	while (cmd[nbargscmd] != NULL) {
		nbargscmd ++;
	}
	
	// allouer la mémoire pour le tableau d'arguments
	j->cmd = malloc((nbargscmd + 1) * sizeof(char *));

	// copier les arguments
	for (int i = 0; i < nbargscmd; i++) {
		j->cmd[i] = strdup(cmd[i]);
	}

	// ajouter NULL à la fin du tableau
	j->cmd[nbargscmd] = NULL;

	if (jobs == NULL) {

     jobs = malloc(sizeof(job)); 
	 // Allouer la mémoire pour le premier processus

  } else {

  	jobs = realloc(jobs, (nbjobs + 1) * sizeof(job)); 
	// Redimensionner la mémoire pour ajouter un nouveau processus

  }

	// Ajouter le processus à la liste
  jobs[nbjobs] = *j; 
	nbjobs++;
}

/* 
 * Supprime un processus de la liste des processus.
*/
void remove_job(pid_t pid) {

    int index_to_remove = -1;

    // Trouver l'index du processus à supprimer
    for (int i = 0; i < nbjobs; i++) {

        if (jobs[i].pid == pid) {
            index_to_remove = i;
            break;
        }
    }

    // Si l'index n'est pas trouvé, ne rien faire
    if (index_to_remove == -1) {
        return;
    }

	// sinon, supprimer le processus
    // Libérer la mémoire allouée pour le job à supprimer
    free(jobs[index_to_remove].state);

		for (int i = 0; jobs[index_to_remove].cmd[i] != NULL; i++) {
      free(jobs[index_to_remove].cmd[i]);
    }

    free(jobs[index_to_remove].cmd);

    // Décaler tous les éléments suivants vers la gauche pour combler l'espace vide
    for (int i = index_to_remove; i < nbjobs - 1; i++) {
        jobs[i] = jobs[i + 1];
    }

    // Réduire la taille du tableau et mettre à jour le nombre de jobs
    nbjobs--;
		if (nbjobs == 0) {
			free(jobs);
			jobs = NULL;
		} else {
			jobs = realloc(jobs, (nbjobs) * sizeof(job));
  	}
}

/*
 * récupère le process id d'un processus en fonction de son 
 * minishell id
 * 0 si le processus n'existe pas dans cette liste
*/
pid_t get_pid(int id) {

	for (int i = 0; i < nbjobs; i++) {

		if (jobs[i].id == id) {
			return jobs[i].pid;
		}
	}

	return 0;
}

/* 
 * récupère les paramètres d'un processus en fonction de son 
 * id minishell.
 * Retourne un pointeur vers le job correspondant
 * Retourne NULL si aucun job ne correspond au pid donné
*/
job* get_job_id(int id) {

	for (int i = 0; i < nbjobs; i++) {

		if (jobs[i].id == id) {
			return &jobs[i];
		}
	}

	return NULL;
}


/*
 * récupère les paramètres d'un processus en fonction
 * de son process id
 * Retourne un pointeur vers le job correspondant
 * Retourne NULL si aucun job ne correspond au pid donné
*/
job* get_job(pid_t pid) {

    for (int i = 0; i < nbjobs; i++) {

        if (jobs[i].pid == pid) {
            return &jobs[i]; 
        }
    }

    return NULL;
}

```

Main and functions :
==
```c
/*
 * GAUTHIER Nino
 * 1A - J - SN
 * 26/04/2023
 * Rendu final du projet minishell.
 * Toutes les fonctionnalités demandées sont implémentées.
*/ 

// importation des librairies
#define _POSIX_C_SOURCE 200809L

#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/wait.h>
#include <string.h>
#include <signal.h>
#include <sys/utsname.h>
#include <fcntl.h>
#include <libgen.h>
#include <git2.h>
#include "readcmd.h"
#include "job.h"


////////// Définition des constantes //////////

const char * EXIT = "exit"; // Commande de sortie du minishell
const char * PROMPT = "\033[33m%s \033[0min \033[32m%s \033[0min \033[34m%s\033[0m\n➜ "; // Prompt du minishell

////////// Définition des variables globales //////////

// Id minishell du processus en avant-plan
int fg_id = 0;

// Liste des processus
extern job * jobs;


/*
 * sous-programme affichant la prompt du minishell
 * en fonction de l'username, du hostname et du nom du dossier courant
*/
void prompt() {
    // nom de l'utilisateur, du host et du dossier courant.
    char username[256], hostname[256], cwd[1024], *current_dir;

    getlogin_r(username, sizeof(username));
    gethostname(hostname, sizeof(hostname));
    getcwd(cwd, sizeof(cwd));
    current_dir = basename(cwd);

    printf(PROMPT, username, hostname, current_dir);

    fflush(stdout);
}




////////// Gestion des signaux //////////


/*
 * Hanfler du signal SIGCHLD
 * récupère le job associé au processus fils qui se termine
 * le retire de la liste des processus
 * indique à l'utilisateur que ce processus s'est terminé
*/
void sigchld_handler(__attribute__((unused)) int signal) {

	int status;
	pid_t pid;

	// tant qu'il y a des processus fils qui se terminent
	while ((pid = waitpid(-1, &status, WNOHANG)) > 0) {

		job *j = get_job(pid);

		// si le processus est dans la liste des processus
		if (j != NULL) {

			int idd = j->id;
			char *cmdd = strdup(j->cmd[0]);

			// le supprimer de la liste des processus
			remove_job(pid);

			printf("\033[33m(-) \033[0mLe job %d (%s) s'est terminé avec le status %d\n", idd, cmdd, WEXITSTATUS(status));

			if (fg_id <= 0) {
				prompt();
			}

			fflush(stdout);
			
		}

		if (pid == get_pid(fg_id)) {
			fg_id = 0;
		}
	}
}


/*
 * Handler du signal SIGINT
 * intercepte ce signal
 * suspend le processus en avant-plan s'il en existe un
*/
void sigint_handler(__attribute__((unused)) int signal) {

		printf("\n");

		// si un processus est en avant-plan
		if (fg_id != 0) {

			// on le termine
			pid_t fg_pid = get_pid(fg_id);
			kill(fg_pid, SIGTERM);

			printf("\033[33m(-) \033[0mprocess terminated\n");
			fflush(stdout);
		} else {
			prompt();
		}
}


/*
 * Handler du signal SIGSTP
 * Arrete le processus en avant-plan
*/
void sigtstp_handler(__attribute__((unused)) int signal) {

		printf("\n");

		// si un processus est en avant-plan
		if (fg_id != 0) {

			// on le suspend
			pid_t fg_pid = get_pid(fg_id);
			kill(fg_pid, SIGSTOP);

			printf("\033[33m(-) \033[0mprocess suspendu\n");
			fflush(stdout);

		} else {

			prompt();

		}
}


////////// Gestion des redirections standards //////////


/*
 * Redirection de la sortie standard vers un fichier
 * si un fichier de sortie est spécifié dans la ligne
 * de commande.
*/
void output_redirect(char *output_file) {

	// Si un fichier de sortie est spécifié
	if (output_file != NULL) {

		// Ouverture du fichier en écriture seule
        int fd = open(output_file, O_WRONLY | O_CREAT | O_TRUNC, 00700);

        if (fd == -1) {

            perror(output_file);
            exit(EXIT_FAILURE);

        }

		// Redirection de la sortie standard vers le fichier
        dup2(fd, STDOUT_FILENO);
        close(fd);
    }
}


/*
 * Redirection de l'entrée standard vers un fichier
 * si un fichier d'entrée est spécifié dans la ligne
 * de commande.
*/
void input_redirect(char *input_file) {

	// Si un fichier d'entrée est spécifié
	if (input_file != NULL) {

		// Ouverture du fichier en lecture seule
		int fd = open(input_file, O_RDONLY);

    if (fd < 0) {
        perror("\033[31m(!) \033[0mErreur lors de l'ouverture du fichier");
        exit(EXIT_FAILURE);
    }

	// Redirection de l'entrée standard vers le fichier
    if (dup2(fd, 0) < 0) {
        perror("\033[31m(!) \033[0mErreur lors de la redirection de l'entrée standard");
        exit(EXIT_FAILURE);
    }

    close(fd);
	}
}


////////// Commandes internes //////////

/*
 * Commande interne suspendant le minishell
*/ 
void susp() {
	printf("\033[31m(!) \033[0mminishell suspendu\n");

	// Envoi du signal SIGSTOP au processus courant
	if (kill(getpid(), SIGSTOP) == -1) {
    perror("\033[31m(!) \033[0mErreur lors de l'envoi du signal SIGSTOP");
    exit(1);
  }

}

/*
 * Commande interne modifiant l'espace de travail du minishell
*/
void cd(char * path) {
   	chdir(path);
	return;
}

/*
 * Commande interne terminant le minishell.
*/
void exit_shell() {

    printf("Goodbye!\n");
    exit(EXIT_SUCCESS);
	return;
}

/*
 * Commande interne listant les processus en cours 
*/
void lj() {

	// Nombre de processus en cours
	extern int nbjobs;	

	// Si aucun processus en cours
	if (nbjobs == 0) {
		printf("Aucun processus en cours. \n");
		return;
	}

	// Affichage des processus en cours
	for (int i = 0; i < nbjobs; i++) {

		printf("Minishell id : %d | PID : %d | Statut : %s | Commande : ", jobs[i].id, (int)jobs[i].pid, jobs[i].state);

		for (int j = 0; jobs[i].cmd[j] != NULL; j++) {
			printf("%s ", jobs[i].cmd[j]);
	}
		printf("\n");
	}

}

/*
 * Commande interne suspendant un processus en cours
*/
void sj(int id) {

	job* j = get_job_id(id);

	if(j == NULL) {

		printf("\033[31m(!) \033[0mCe processus n'existe pas\n");

	} else {
		
		// Passer l'état à "suspended"
		free (j->state);
		j->state = strdup("suspended");

		// Envoyer un signal SIGSTOP pour suspendre le processus
		if (kill(j->pid, SIGSTOP) == -1) {

			perror("\033[31m(!) \033[0mErreur à l'envoi de SIGSTOP");
			exit(EXIT_FAILURE);

		}
	}
}

/*
 * Commande interne reprenant en arrière plan un processus
 * suspendu
*/
void bg(int id) {

	job* j = get_job_id(id);

	if(j == NULL) {

		printf("\033[31m(!) \033[0mCe processus n'existe pas\n");

	} else {
		
		// Passer l'état à "actif"
		free(j->state);
		j->state = strdup("actif");

		// Envoyer un signal SIGCONT pour reprendre le processus s'il était suspendu
		if (kill(j->pid, SIGCONT) == -1) {

			perror("\033[31m(!) \033[0mErreur à l'envoi de SIGCONT");
			exit(EXIT_FAILURE);

		}
	}
}

/*
 * Commande interne reprenant en avant plan un processus suspendu.
*/
void fg(int id, int *wait_fg) {

    job* j = get_job_id(id);

    if (j == NULL) {
        printf("\033[31m(!) \033[0mCe processus n'existe pas\n");
    } else {

      // Passer l'état à "actif"
      free(j->state);
      j->state = strdup("actif");

      // Envoyer un signal SIGCONT pour reprendre le processus s'il était suspendu
      if (kill(j->pid, SIGCONT) == -1) {
        perror("\033[31m(!) \033[0mErreur à l'envoi de SIGCONT");
        exit(EXIT_FAILURE);
      }

			fg_id = id;
			*wait_fg = 1;
    }
}


////////// Execution des commandes internes //////////

/*
 * Execution des commandes internes en fonction du 
 * premier argument de la ligne de commande.
*/
void exec_cmd_interne(char *args[], int *cmdInterne, int *wait_fg) {
	if (strcmp(args[0], "cd") == 0) {
		*cmdInterne = 1;
		cd(args[1]);
	} else if (strcmp(args[0], "exit") == 0) {
		*cmdInterne = 1;
		exit_shell();
	} else if (strcmp(args[0], "lj") == 0) {
		*cmdInterne = 1;
		lj();
	} else if (strcmp(args[0], "sj") == 0) {
		*cmdInterne = 1;
		sj(atoi(args[1]));
	} else if (strcmp(args[0], "bg") == 0) {
		*cmdInterne = 1;
		bg(atoi(args[1]));
	} else if (strcmp(args[0], "fg") == 0) {
		*cmdInterne = 1;
		fg(atoi(args[1]), wait_fg);
	} else if (strcmp(args[0], "susp") == 0) {
		*cmdInterne = 1;
		susp();
	}
}

////////// Analyse de la ligne de commande //////////

/*
 * Analyse de la ligne de commande.
*/
void analyse_cmd(struct cmdline *line, int *cmdBackground, int *nbcmd, int **nbargs) {

	*cmdBackground = 0;
	if (line->backgrounded != NULL && strcmp(line->backgrounded, "&") == 0){
		*cmdBackground = 1;
	}

	// calcul du nombre de commandes

	*nbcmd = 0;
	while ((line->seq)[*nbcmd] != NULL){
		(*nbcmd)++;
	}

	// calcul du tableau contenant le nombre d'arguments par commande

	*nbargs = NULL;
	*nbargs = realloc(*nbargs, (*nbcmd+1)*sizeof(int));
	for (int i = 0; i < *nbcmd; i++){
		(*nbargs)[i] = 0;
		int j = 0;
		while ((line->seq)[i][j] != NULL){
			(*nbargs)[i]++;
			j++;
		}
	}
}

////////// Sous-programmes père et fils //////////


/*
 * Sous-programme créant un processus fils 
 * ayant pour rôle d'exécuter la commande.
*/
void creer_fils(char *sequence[], int cmdBackground, pid_t *pidFils, int *wait_fg) {

	// compteur pour définir les id des processus
	extern int job_id_counter;

	job_id_counter++;

	if (!cmdBackground){
	*wait_fg = 1;
	fg_id = job_id_counter;}

	
	*pidFils = fork();
	
	add_job(job_id_counter, *pidFils, "actif", sequence);

	if (*pidFils == -1) {
		perror("\033[31m(!) \033[0mErreur lors de la création du fils");
		exit(EXIT_FAILURE);
	}

}

/*
 * Sous-programme exécuté par le processus fils.
*/
void fils(char *sequence[], int *nbargs, struct cmdline *line, int k, int nbcmd) {

	//setpgid(0, 0);

	char **args = NULL; // création d'un tableau contenant uniquement les arguments de la premiere commande 
	args = realloc(args, (nbargs[0]+1)*sizeof(char *));

	for (int i = 0; i<nbargs[0]; i++){ // ajout des arguments au tableau 
		args[i] = sequence[i];
	}

	args[nbargs[0]] = NULL; // dernier élément NULL 

	// redirections

	if (k == (nbcmd-1)) {
		output_redirect(line->out); // redirige uniquement la sortie de la dernière commande
	}

	if (k == 0) {
		input_redirect(line->in); // redirige uniquement l'entrée de la première commande
	}

	execvp(args[0], args); // exécution de la première commande seulement
	perror("\033[31m(!) \033[0mErreur lors de l'execution de la commande");
	exit(3);
}

/*
 * Sous-programme exécuté par le processus père.
*/
void pere_fg(int cmdInterne, int *wait_fg) {

	pid_t idFils;

	// code de retour de la commande
	int codeTerm;

	// le dernier processus lancé est celui en avant plan
	
	if (*wait_fg && fg_id > 0) {
		job *j = get_job_id(fg_id);
		if (j != NULL) {
			idFils = waitpid(j->pid, &codeTerm, 0);
		}
	} 
	
	// Réinitialisation des variables d'avant plan 
	*wait_fg = 0;
	fg_id = 0;

	if (idFils == -1 && cmdInterne == 0) {
		perror("\033[31m(!) \033[0mwait ");
		exit(2);
	}

	// on retire le processus de la liste car il est fini.
	if (WEXITSTATUS(codeTerm) == 3) {
		remove_job(idFils);
		//printf("ECHEC\n");
		
	} else {
		remove_job(idFils);
		//printf("SUCCES\n");
		
	}

}

/*
 * Sous-programme exécuté par le processus père lorsque la commande est lancée en arrière plan.
*/
void pere_bg(pid_t pidFils, struct cmdline *line) {

	// compteur pour définir les id des processus
	extern int job_id_counter;

	// ici on ne fait rien, le processus fils est lancé en arrière plan
	printf("\033[32m(+) \033[0mLe job %d (%s) a été ajouté en background avec le PID %d\n", job_id_counter, (line->seq)[0][0], pidFils);

}

////// Gestion des pipes //////

/*
 * Sous-programme créant les pipes.
*/
void init_pipes(int nbcmd, int pipes[][2], int k) {

	// création des pipes
	if (k < nbcmd-1) {

		if (pipe(pipes[k]) < 0) {
			perror("pipe");
			exit(1);
		}
	}
}

/*
 * Redirection des entrées standard.
*/
void input_pipes_redirect(int pipes[][2], int k) {

	if (k != 0) {
		dup2(pipes[k-1][0], STDIN_FILENO);
		close(pipes[k-1][0]);
		close(pipes[k-1][1]);
	}
}

/*
 * Redirection des sorties standard.
*/
void output_pipes_redirect(int pipes[][2], int k, int nbcmd) {

	if (k != nbcmd-1) {
		dup2(pipes[k][1], STDOUT_FILENO);
		close(pipes[k][0]);
		close(pipes[k][1]);
	}
}

/*
 * Fermeture des pipes.
*/
void close_pipes(int pipes[][2], int k) {

	if (k != 0) {
        close(pipes[k - 1][0]);
        close(pipes[k - 1][1]);
    }
}

////////// Programme principal //////////

int main(__attribute__((unused)) int argc, __attribute__((unused)) char *argv[]){

	//// Initialisation des variables ////
	
	// Flag déterminant si la commande doit être traitée en interne
	int cmdInterne = 0;

	// Flag déterminant si la commande doit être lancée en arrière plan
	int cmdBackground = 0;

	// Ligne de commande à traiter
	struct cmdline *line;

	// pid du processus fils
	pid_t pidFils = -1;

	// flag pour savoir si l'on doit attendre le processus en avant-plan
	int wait_fg = 0;

	// nombre de commandes dans la ligne (pipelines)
	int nbcmd = 0;

	// tableau contenant le nombre d'arguments par commande
	// C'est un tableau pour gêrer les différentes commandes des pipelines
	int * nbargs = NULL;




	//// gestion des signaux ////

	struct sigaction sa;

	sa.sa_handler = sigtstp_handler;
	sigemptyset(&sa.sa_mask);
	sa.sa_flags = SA_RESTART;
	sigaction(SIGTSTP, &sa, NULL);

	sa.sa_handler = sigint_handler;
	sigemptyset(&sa.sa_mask);
	sa.sa_flags = SA_RESTART;
	sigaction(SIGINT, &sa, NULL);

	sa.sa_handler = sigchld_handler;
	sigemptyset(&sa.sa_mask);
	sa.sa_flags = SA_RESTART;
	sigaction(SIGCHLD, &sa, NULL);
	

	//// Boucle principale ////

	do {
		

		prompt();
		fflush(stdout);

		line = readcmd(); // Interprète et renvoie la ligne de commande dans line

		// Si la ligne entrée est vide, on n'effectue aucune action.
		if (line != NULL) {

			//// Initialisation des structures de données liées aux arguments ////

			analyse_cmd(line, &cmdBackground, &nbcmd, &nbargs);

			
			// création du tableau contenant les pipes
			int pipes[nbcmd - 1][2];

			for (int k = 0; k < nbcmd; k++) {

				init_pipes(nbcmd, pipes, k);

				//// Execution des commandes internes ////
				// si la commande est interne, on l'execute dans le processus courant
				cmdInterne = 0;
				exec_cmd_interne((line->seq)[k], &cmdInterne, &wait_fg);

				// si la commande n'est pas interne, on la lance dans un processus fils

				if (cmdInterne == 0) {
					creer_fils((line->seq)[k], cmdBackground, &pidFils, &wait_fg);
				}

				if (pidFils == 0){ /* fils */	
				
					// Redirections des entrées et sorties
					
					input_pipes_redirect(pipes, k);

					output_pipes_redirect(pipes, k, nbcmd);

					// Execution du programme fils
					fils((line->seq)[k], nbargs, line, k, nbcmd);

				} else {/* père */

					// fermeture des pipes des commandes précédentes
					
					close_pipes(pipes, k);

					// Si la commande est interne, nous ne la traitons pas dans le père

					if (!cmdInterne || wait_fg) { 
						
						if (!cmdBackground) {

						// Traitement des commandes en premier plan

							pere_fg(cmdInterne, &wait_fg);

						} else {

							// Traitement des commandes en arrière plan

							pere_bg(pidFils, line);

						}
					}
				}
			}
		}
	} while (line != NULL);
	// Fin de la boucle principale lorsque l'utilisateur tape Ctrl+D
	
	printf("\nGoodbye!\n");
	return EXIT_SUCCESS;
}
```


