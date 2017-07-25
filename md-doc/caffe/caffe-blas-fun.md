常用 blas 函数

Y=alpha * X +beta*Y 

template <>
void caffe_cpu_axpby<float>(const int N, const float alpha, const float* X,
                            const float beta, float* Y) {
  cblas_saxpby(N, alpha, X, 1, beta, Y, 1);
}

template <>
void caffe_cpu_axpby<double>(const int N, const double alpha, const double* X,
                             const double beta, double* Y) {
  cblas_daxpby(N, alpha, X, 1, beta, Y, 1);
}
复制代码
 
cblas_dscal(N, beta, Y, incY);  Y=Y*beta 
cblas_daxpy(N, alpha, X, incX, Y, incY);  Y= (alpha * X) + Y)

复制代码
 

Y=alpha * X + Y 

复制代码
template <>
void caffe_axpy<float>(const int N, const float alpha, const float* X,
    float* Y) { cblas_saxpy(N, alpha, X, 1, Y, 1); }

template <>
void caffe_axpy<double>(const int N, const double alpha, const double* X,
    double* Y) { cblas_daxpy(N, alpha, X, 1, Y, 1); }
复制代码
复制代码
DEFINE_VSL_BINARY_FUNC(Add, y[i] = a[i] + b[i]);
DEFINE_VSL_BINARY_FUNC(Sub, y[i] = a[i] - b[i]);
DEFINE_VSL_BINARY_FUNC(Mul, y[i] = a[i] * b[i]);
DEFINE_VSL_BINARY_FUNC(Div, y[i] = a[i] / b[i]);


template <>
void caffe_add<float>(const int n, const float* a, const float* b,
float* y) {
vsAdd(n, a, b, y);
}

template <>
void caffe_add<double>(const int n, const double* a, const double* b,
double* y) {
vdAdd(n, a, b, y);
}
复制代码
 

y=x;

复制代码
template <>
void caffe_copy<float>(const int N, const float* X, float* Y) {
  cblas_scopy(N, X, 1, Y, 1);
}

template <>
void caffe_copy<double>(const int N, const double* X, double* Y) {
  cblas_dcopy(N, X, 1, Y, 1);
}

template <>
void caffe_gpu_copy<float>(const int N, const float* X, float* Y) {
  CUBLAS_CHECK(cublasScopy(Caffe::cublas_handle(), N, X, 1, Y, 1));
}

template <>
void caffe_gpu_copy<double>(const int N, const double* X, double* Y) {
  CUBLAS_CHECK(cublasDcopy(Caffe::cublas_handle(), N, X, 1, Y, 1));
}
复制代码
 

Computes alpha*x*y' + A.

复制代码
cblas_sger
Multiplies vector X by the transform of vector Y, then adds matrix A (single precison).
Multiplies vector X by the transform of vector Y, then adds matrix A (single precison).
void cblas_sger (
const enum CBLAS_ORDER Order,
const int M,
const int N,
const float alpha,
const float *X,
const int incX,
const float *Y,
const int incY,
float *A,
const int lda
);

复制代码
复制代码
Y(vetor)←αAX + βY

This function multiplies A * X (after transposing A, if needed) and multiplies the resulting matrix by alpha.
It then multiplies vector Y by beta. It stores the sum of these two products in vector Y.

template <>
void caffe_cpu_gemv<float>(const CBLAS_TRANSPOSE TransA, const int M,
    const int N, const float alpha, const float* A, const float* x,
    const float beta, float* y) {
  cblas_sgemv(CblasRowMajor, TransA, M, N, alpha, A, N, x, 1, beta, y, 1);
}
复制代码
 

C(matrix)←αAB + βC

复制代码
template<typename T>
void gpu_multmat(T* A, T* B, T* C, int M,int K,int N){
     const T alpha = 1,beta=0;
     caffe_gpu_gemm(CblasNoTrans,CblasNoTrans,M,N,K,alpha,A,B,beta,C);
}
template<>
void caffe_cpu_gemm<float>(const CBLAS_TRANSPOSE TransA,
    const CBLAS_TRANSPOSE TransB, const int M, const int N, const int K,
    const float alpha, const float* A, const float* B, const float beta,
    float* C) {
  int lda = (TransA == CblasNoTrans) ? K : M;
  int ldb = (TransB == CblasNoTrans) ? N : K;
  cblas_sgemm(CblasRowMajor, TransA, TransB, M, N, K, alpha, A, lda, B,
      ldb, beta, C, N);
}
复制代码
 

复制代码
A=M*N  B=M*K
C=A'*B   N M K

template<typename T>
void cpu_multTmat(T* A, T* B, T* C, int M,int K,int N){
     const T alpha = 1,beta=0;
     caffe_cpu_gemm(CblasTrans,CblasNoTrans,M,N,K,alpha,A,B,beta,C);
    // cblas_dgemm(CblasRowMajor, CblasNoTrans, CblasNoTrans, M, N, K, alpha, A, M, B,    K, beta, C, M);
}
A=M*N B=N*K
C=A*B   M N K


template<typename T>
void cpu_multmat(T* A, T* B, T* C, int M,int K,int N){
     const T alpha = 1,beta=0;
     caffe_cpu_gemm(CblasNoTrans,CblasNoTrans,M,N,K,alpha,A,B,beta,C);
    // cblas_dgemm(CblasRowMajor, CblasNoTrans, CblasNoTrans, M, N, K, alpha, A, M, B,    K, beta, C, M);
}
