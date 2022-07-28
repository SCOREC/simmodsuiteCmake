cmake_minimum_required(VERSION 3.20)

project(foo VERSION 1.0.0 LANGUAGES CXX)

find_package(simmetrix-simmodsuite REQUIRED)

add_executable(testSim testSim.cpp)
target_link_libraries(testSim simmetrix-simmodsuite::SimModSuite)
