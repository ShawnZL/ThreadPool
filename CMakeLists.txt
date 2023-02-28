cmake_minimum_required(VERSION 3.21)
project(ThreadPool C)

set(CMAKE_C_STANDARD 23)

add_executable(ThreadPool main.c threadpool.h)
