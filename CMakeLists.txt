set(TENSORFLOW_LITE_LIB  
  "${CMAKE_CURRENT_SOURCE_DIR}/tflite/libtensorflow-lite.a"
  "${CMAKE_CURRENT_SOURCE_DIR}/tflite/libclog.a"
  "${CMAKE_CURRENT_SOURCE_DIR}/tflite/libcpuinfo.a"
  "${CMAKE_CURRENT_SOURCE_DIR}/tflite/libfarmhash.a"
  "${CMAKE_CURRENT_SOURCE_DIR}/tflite/libfft2d_fftsg.a"
  "${CMAKE_CURRENT_SOURCE_DIR}/tflite/libfft2d_fftsg2d.a"
  "${CMAKE_CURRENT_SOURCE_DIR}/tflite/libflatbuffers.a"
  "${CMAKE_CURRENT_SOURCE_DIR}/tflite/libpthreadpool.a"
  "${CMAKE_CURRENT_SOURCE_DIR}/tflite/libruy.a"
  "${CMAKE_CURRENT_SOURCE_DIR}/tflite/libXNNPACK.a"
  CACHE INTERNAL "${TENSORFLOW_LITE_LIB}"
)
set(TENSORFLOW_LITE_INCLUDE
  "${CMAKE_CURRENT_SOURCE_DIR}"
  CACHE INTERNAL "${TENSORFLOW_LITE_INCLUDE}"
)
