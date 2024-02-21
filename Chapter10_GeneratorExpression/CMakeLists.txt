cmake_minimum_required(VERSION 3.8)
project(generator_expressiong)

add_executable(myApp src1.cpp src2.cpp)

# Before CMake 3.8
target_link_libraries(myApp PRIVATE
    $<$<CONFIG:Debug>:CheckedAlgo>
    $<$<NOT:$<CONFIG:Debug>>:FastAlgo>
)

# After CMake 3.8
target_link_libraries(myApp PRIVATE $<IF:$<CONFIG:Debug>, CheckedAlgo, FastAlgo>)