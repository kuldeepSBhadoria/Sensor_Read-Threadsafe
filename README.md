# Sensor_Read-Threadsafe
#include <stdio.h>
#include <stdbool.h>
#include <stdlib.h>
#include <stdint.h>
#include <sys/time.h>
#include <unistd.h>
#include <pthread.h>
#include <iostream>
#include <deque>
#include<semaphore.h>

/*
Light sensor question
Background
==========
Security cameras often have an Ambient Light Sensor (ALS) which measures the amount of light around the camera and triggers a "night mode" when the ambient light is too low.

You've been given an ALS that reports sensor readings using the following struct:
  typedef struct SensorReading {
    int status;
    float lux;
    uint64_t timestamp; // time of reading
  } SensorReading;


To get a sensor reading, call:
SensorReading read_next_sample(uint64_t max_wait) { ... }

This function is blocking!  It doesn't return a value until either max_wait microseconds have elapsed or the ALS returns a new value.  The SensorReading.status int indicates whether the value changed (VALID) or not (NO_CHANGE).

(We've implemented a mock version of read_next_sample below, but you do not need to read the code and understand it to complete this task).

Task
====
Working with blocking functions is hard and not thread-friendly, so we want to wrap the ALS function in a non-blocking, thread-safe way.
Design and implement an API that allows users to read lux values from any time within the last 10 minutes.  Your API should be non-blocking and thread-safe.
*/

/* Status enum */
enum {
  ERROR = 0,
  NO_CHANGE = 1,
  VALID = 2,
};
#define Th_cnt 10
#define Q_size 10

/* Don't edit this code */
typedef struct SensorReading {
  int status;
  float lux;
  uint64_t timestamp;
} SensorReading;

pthread_mutex_t writeMtx = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t readMtx = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
sem_t wrt;

volatile uint64_t timeStampCheck;
int readCount = 0;

/*! Don't edit this code */
uint64_t get_timestamp();

struct SensorReading read_next_sample(uint64_t max_wait);
std::deque<SensorReading> Q;

float get_lux_value(uint64_t timestamp){
    for (auto it = Q.begin(); it != Q.end(); ++it) {
        if (it->timestamp == timestamp) {
            return it->lux;
        }
    }
    // Should not block
    return -1.0;
}

void *readSample(void *argv) {
    int threadNum = (int)*(int *)argv;
    while(1) {
        
       // sem_wait(&wrt);
       // sem_post(&wrt);
        
        // Incremnet the count of number of readers when entering the read thread
        pthread_mutex_lock(&readMtx);
        readCount++;
        if(readCount == 1) {
            sem_wait(&wrt); // If this id the first reader, then it will block the writer
        }
        pthread_mutex_unlock(&readMtx);
        
        printf("threadNum: %d, lux: %f\n", threadNum, get_lux_value(timeStampCheck));
        
        pthread_mutex_lock(&readMtx);
        readCount--;
        
        if(readCount == 0) {
             sem_post(&wrt); // If this is the last reader, it will wake up the writer.
         }
        pthread_mutex_unlock(&readMtx);
        
        usleep(1000000);
        
    }
}

void *writeSample(void *argv) {
    bool startWrite = true;
      struct timespec ts;
    while(1) {
        // read sample every 10 secs
        struct SensorReading reading = read_next_sample(10 * 1000000);
        timeStampCheck = reading.timestamp;
  
        bool done = false;
        while(!done) {
           // sem_wait(&wrt);
            if (clock_gettime(CLOCK_REALTIME, &ts) == -1) {
                        printf("clock_gettime error\n");
                    }
                ts.tv_sec += 0.2;
                 if (sem_timedwait(&wrt, &ts) == -1) {
                        printf("TimeOUT\n");
                    }
            if (Q.size() >= Q_size) 
                Q.erase(Q.begin());
        
                // store the samples in queue which holds 10 mins of data. 
                // 10 mins = 600 secs / 10 secs..wo quue should hold atleast 60 samples
                Q.push_back(reading);
                printf("lux: %02f, ts: %lu, status: %d\n", reading.lux, reading.timestamp, reading.status);
                 usleep(3000000);
                done = true;   
            sem_post(&wrt);
        }
            
    }
}

int main() {
    
    pthread_t writeThread;
    int retWriteSample = pthread_create(&writeThread, NULL, writeSample, NULL);
    sem_init(&wrt,0,1);

    pthread_t readThread[Th_cnt];
    int *arr = (int *) malloc(sizeof(int) * Th_cnt);
    for(int i = 0; i < Th_cnt; i++) {
        arr[i] = i;
        pthread_create(&readThread[i], NULL, readSample, (void *)&arr[i]);
    }
    
    for(int i = 0; i < Th_cnt; i++) {
        pthread_join(readThread[i], NULL);
    }
    
    if (!retWriteSample) {
        pthread_join(writeThread, NULL);
    }
      
    return 0;
}





/***** You _don't_ need to read the following code to complete the exercise; it's the implementation of the interfaces described above *****/

/* Don't edit this code */
float random_float() {
    return (float)rand() / RAND_MAX; 
}

/*! Don't edit this code */
uint64_t get_timestamp() {
  struct timeval tv;

  gettimeofday(&tv, NULL);
  return 1000000 * tv.tv_sec + tv.tv_usec;
}

/* Don't edit this code */
struct SensorReading read_next_sample(uint64_t max_wait) {
  static bool first_run = true;
  struct SensorReading return_value;

  if (random_float() < 0.0001) {
    return_value.status = ERROR;
    return return_value;
  }

  if (first_run) {
    first_run = false;
    return_value.lux = random_float() * 10;
    return_value.status = VALID;
    return_value.timestamp = get_timestamp();
    return return_value;
  }

  uint64_t next_value_change = 30 * 1000000 * random_float();

  if (max_wait <= next_value_change) {
    usleep(max_wait);
    return_value.status = NO_CHANGE;
    return return_value;
  }

  usleep(next_value_change);
  return_value.lux = random_float() * 10;
  return_value.status = VALID;
  return_value.timestamp = get_timestamp();
  return return_value;
}


