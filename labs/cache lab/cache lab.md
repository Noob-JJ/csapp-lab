```c
#include "cachelab.h"
#include <stdio.h>
#include <math.h>
#include <string.h>
#include <stdlib.h>

static int hits = 0;
static int misses = 0;
static int evictions = 0;

//if print detail info;
static int h = 0;
static int v = 0;

static char *tracefile;

static int s;
static int S;
static int E;
static int b;
static int B;

struct line{
    int valid;
    int flag;
    struct line *pre;
    struct line *next;
};

struct set{
    int nowNumber;
    struct line *header;
    struct line *tail;
};


static struct set *sets;


int cli(int agrc, char *argv[]);
int initStruct();
int getT(int address);
int getS(int address);
void handle();
int getLine(FILE *fp, char* buffer);
void handleOne(char *ch);
int hexToTen(char *hex);

int main(int agrc, char *argv[])
{
    cli(agrc, argv);
    initStruct();
    handle();
    printSummary(hits, misses, evictions);
    return 0;
}


int cli(int agrc, char *argv[]){
    int index = 1;
    while(index < agrc){
        char *param = argv[index];

        if(strcmp(param, "-h") == 0){
            h = 1;
        }
        else if(strcmp(param, "-v") == 0){
            v = 1;
        }
        else if(strcmp(param, "-s") == 0){
            index++;
            char *sString = argv[index];
            s= atoi(sString);
            S = pow(2, s);
        }
        else if(strcmp(param, "-E") == 0){
            index++;
            char *EString = argv[index];
            E = atoi(EString);
        }
        else if(strcmp(param, "-b") == 0){
            index++;
            char *bString = argv[index];
            b = atoi(bString);
            B = pow(2, b);
        }
        else if(strcmp(param, "-t") == 0){
            index++;
            tracefile = argv[index];
        }
        else{
            printf("runtime param error, pleace re-run again\n");
        }
        index++;
    }
}

int initStruct(){
    sets = malloc(S * sizeof(struct set));
}


int getT(int address){
    return (unsigned int)address >> (s + b);
}

int getS(int address){
    return (unsigned int)(address >> b) & (S - 1);
}

int closeProcess(){

}

void handle(){
    FILE *fp = fopen(tracefile, "r");
    char buffer[22];
    while(getLine(fp, buffer) != 0){
        handleOne(buffer);
    }
}

int getLine(FILE *fp, char* buffer){
    int i = 0;
    while(1){
        char ch = getc(fp);
        if(ch == '\n'){
            buffer[i] = '\0';
            return 1;
        }
        else if(ch == EOF){
            return 0;
        }
        else{
            buffer[i++] = ch;
        }
    }
}

void handleOne(char *ch){
    char instruct = ch[0];
    if(instruct == 'I'){
        return;
    }
    char mode = ch[1];

    char *chback = ch + 3;
    char address[17];

    if(v == 1){
        printf("%s", ch);
    }

    int executeTimes = mode == 'M' ? 2 : 1;

    int i = 0;
    while((*chback) != '\0' && (*chback) != ','){
        address[i++] = *chback;
        chback++;
    }
    address[i] = '\0';
 
    int addressInt = hexToTen(address);

    int n_flag = getT(addressInt);
    int n_s = getS(addressInt);

    struct set *n_set = sets + n_s;

    for(int i = 0; i < executeTimes; i++){
        struct line *first = n_set -> header;
        struct line *index = first;
        int find = 0;
        while(index != NULL){
            if(index -> flag == n_flag && index -> valid == 1){
                find = 1;
                break;
            }
            index = index -> next;
        }

        if(find == 1){

            if(first != index){
                if(index == n_set -> tail){
                    n_set-> tail = index -> pre;
                }
                else {
                index -> next -> pre = index -> pre;
                }

                index -> pre -> next = index -> next;
                index -> pre = NULL;
                index -> next = first;
                first -> pre = index;
                first = index;
                n_set -> header = index;
                //todo index = tail handler
            }
            hits++;
            if(v == 1){
                printf(" hit");
            }
        }
        else {
            if(n_set -> nowNumber < E){
                struct line *newLine = malloc(sizeof(struct line));
                newLine -> flag = n_flag;
                newLine -> valid = 1;

                if(first == NULL){
                    n_set -> header = newLine;
                    n_set -> tail = newLine;
                }
                else {
                    newLine -> next = first;
                    first -> pre = newLine;
                    n_set -> header = newLine;
                    first = newLine;
                }

                n_set -> nowNumber = n_set -> nowNumber + 1;

                misses++;

                if(v == 1){
                    printf(" miss");
                }
            }
            else 
            {

                struct line *n_tail = n_set -> tail;
                n_tail -> flag = n_flag;
                n_tail -> valid = 1;
                if(first != n_tail){
                    n_set -> tail = n_tail -> pre;
                    n_tail -> pre -> next = NULL;
                    n_tail -> pre = NULL;
                    n_tail -> next = first;
                    first -> pre = n_tail;
                    n_set -> header = n_tail;
                    first = n_tail;
                }

                misses++;
                evictions++;
                
                if(v == 1){
                    printf(" miss eviction");
                }
            }
        }
    }
    printf("\n");
}


int hexToTen(char *hex){
    int sum = 0;

    while(*hex != '\0'){
        char ch = *hex;
        int t;
        if(ch >= '0' && ch <= '9'){
            t = ch - '0';
        }
        else if(ch >= 'a' && ch <= 'f'){
            t = 10 + ch - 'a';
        }
        else if(ch >= 'A' && ch <= 'F'){
            t = 10 + ch - 'A';
        }
        else{
            printf("address error, process exit\n");
            exit(1);
        }
        sum = sum * 16 + t;
        hex++;
    }

    return sum;
}
```

