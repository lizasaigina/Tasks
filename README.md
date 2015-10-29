#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <pthread.h>
#include <sys/sem.h>
#include <sys/ipc.h>


// IPC-дескриптор для массива семафоров,
// который содержит единственный мьютекс, доступный потокам
int globalSemId;
// глоб. переменная, именяемая потоками
unsigned long globalIncrementedValue = 0;


struct thread_info {
    pthread_t thread_id;     // ID returned by pthread_create()
    unsigned int thread_num; // Application-defined thread number
};

void* worker_thread(void* param);

#define WORKER_THEAD_WORKLOAD 100000000l


int main()
{
    struct thread_info* threadInfo;
    int res, threadTotalNumber;


    //инициализация глоб. мьютекса
    {
        // IPC ключ
        key_t key;

        // Файл с именем "file4ipckey1" должен существовать в workspace директории
        if ( (key = ftok("../../file4ipckey1", 0)) < 0 ) {
            perror("Can`t generate key");
            exit(EXIT_FAILURE);
        }

        // Пытаемся получить доступ по ключу к массиву семафоров, если он существует,
        // или создать его из одного семафора, если его ещё не существует, с правами
        // доступа rw для всех пользователей
        if ((globalSemId = semget(key , 1 , IPC_CREAT | 0666 )) < 0) {
            perror("Can`t get globalSemId");
            exit(EXIT_FAILURE);
        }

        // после создания семафора его состояние равно 0, повышаем состояние до 1,
        // чтобы он мог использоваться в кач-ве мьютекса для дочерних потоков
        struct sembuf buf;
        buf.sem_op = 1;
        buf.sem_flg = 0;
        buf.sem_num = 0;
        if (semop(globalSemId , &buf , 1) < 0) {
            perror("Can`t wait for condition\n");
            exit(-1);
        }
    }

    // пользователь вводит кол-во потоков
    // threadTotalNumber = 5;
    do
    {
        printf("main: enter desired number of threads to create: ");
        res = scanf("%d", &threadTotalNumber);
        fflush(stdin); //не вводить строку! fflush почему-то не работает

        if(res <= 0)
        {
            printf("main: nothing was entered.\n");
        }
        else if( threadTotalNumber <= 0 || threadTotalNumber > 1000 )
        {
            printf("main: %d is invalid, please enter reasonable number from 0 to 1000.\n", threadTotalNumber);
        }
    }
    while( threadTotalNumber <= 0 || threadTotalNumber > 1000 );


    //созданем массив структур, заранее инициализированных null и 0
    threadInfo = calloc(threadTotalNumber, sizeof(struct thread_info));
    if (threadInfo == NULL) {
        printf("Error during calloc.\n");
        exit(EXIT_FAILURE);
    }

    //создаем потоки, заполняя массив threadInfo
    for(int iter=0; iter<threadTotalNumber; iter++) {

        threadInfo[iter].thread_num = iter + 1;

        res = pthread_create( &threadInfo[iter].thread_id
                              , (pthread_attr_t *)NULL
                              , worker_thread
                              , &threadInfo[iter].thread_num );
        if (res) {
            printf("Can`t create thread #%u, returned value = %d\n", threadInfo[iter].thread_num, res);
            free(threadInfo);   //всегда очищаем выделенную нами дин. память
            exit(EXIT_FAILURE); //при завершении всего процесса, дочерние потоки также завершатся
        }

        printf("main: thread #%u [pid=%u] was created.\n", threadInfo[iter].thread_num, (unsigned int)threadInfo[iter].thread_id);
    }

    //ждем пока все созданные ранее потоки отработают
    for (int iter=0; iter<threadTotalNumber; iter++) {

       res = pthread_join( threadInfo[iter].thread_id // по очереди
                           , (void **)NULL );         // потоки ничего не возвращают, иначе пришлось бы очищать память выделенную под возвращенную переменную
       if (res){
            printf("Can`t join thread #%u, returned value = %d.\n", threadInfo[iter].thread_num, res);
            free(threadInfo);
            exit(EXIT_FAILURE);
       }

       printf("main: joined with thread #%u [pid=%u]\n", threadInfo[iter].thread_num, (unsigned int)threadInfo[iter].thread_id);
    }

    //все потоки успешно завершили свою работу, можно проверять результат их работы
    if(globalIncrementedValue == threadTotalNumber*WORKER_THEAD_WORKLOAD) {
        printf("main: globalIncrementedValue = %lu  OK.\n", globalIncrementedValue);
    } else {
        printf("main: globalIncrementedValue = %lu  WARNING: race condition suspected!\n", globalIncrementedValue);
    }


    //удаляем массив семафоров
    if (semctl(globalSemId, 0, IPC_RMID, NULL) == -1) {
        perror("Couldn't delete semaphors with semctl");
        exit(EXIT_FAILURE);
    }


    free(threadInfo);
    exit(EXIT_SUCCESS);
}


void* worker_thread(void* param) {

    pthread_t my_thread_id = pthread_self();
    unsigned int* my_thread_number = param;

    printf("Thread #%u [pid=%u]: starting work.\n", *my_thread_number, (unsigned int)my_thread_id);

    // Инициализация структуры для задания операции над семафором-мьютексом
    struct sembuf buf;
    buf.sem_op = 0;
    buf.sem_flg = 0;
    buf.sem_num = 0;


    // понижаем состояние мьютекса с 1 => 0
    // если текущее состояние мьютекса 0, то ||-ый процесс в крит. области, ждем завершения его работы
    buf.sem_op = -1;
    if (semop(globalSemId , &buf , 1) < 0) {
        perror("Couldn't change semaphor with semop");
        exit(EXIT_FAILURE);
    }


    printf("Thread #%u [pid=%u]: inside critical section; work in progress...\n", *my_thread_number, (unsigned int)my_thread_id);

    // simulating worktime deviation to ensure interleaving of threads
    //if((*my_thread_number)%3 == 0)
    //    sleep(1);

    for(int iter=0; iter<WORKER_THEAD_WORKLOAD; iter++) {

        globalIncrementedValue += 1;
    }


    printf("Thread #%u [pid=%u]: finished work.\n", *my_thread_number, (unsigned int)my_thread_id);

    // возвращаем состояние мьютекса в исходное состояние с 0 => 1
    buf.sem_op = 1;
    if (semop(globalSemId , &buf , 1) < 0) {
        perror("Couldn't change semaphor with semop");
        exit(EXIT_FAILURE);
    }

    return NULL;
}
# Tasks
