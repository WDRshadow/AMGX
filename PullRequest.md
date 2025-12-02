This PR adds a zero-copy mode to the C-API. This allows the user to directly assemble matrices into AmgX, thus avoiding duplicate copies of the matrix data in memory and skipping the reordering of that data that would happen in the normal C-API upload routines. The user takes over responsibility for making sure the matrix layout matches what is used by AmgX internally, i.e. the ordering of rows must conform to the AmgX standard (interior, boundary, halo rows).

In the current implementation, there are additional requirements:

 - (block-)CSR format with embedded diagonals that are placed first in every row only
 - in parallel, halo rows in the input data must contain at least diagonal entries

These additional requirements are not fundamental and the ZC interface could naturally be extended to not depend on them, I did not do so because this is our use case and therefore the only tested configuration.

Additionally, I only tested it with AGGREGATION AMG, but I see no reason why it would not work with CLASSICAL too.

Finally, all data should reside on the GPU as this is completely untested with host matrices.

A zero-copy matrix upload works like this:

 - an AmgX-matrix is created as usual with AMGX_matrix_create
 - AMGX_ZC_matrix_initialize is called on it
 - row-offsets, columns and values are allocated and filled using AMGX_ZC_matrix_data_vec_resize/get_ptr
 - in parallel, communication tables are supplied via the standard AMGX_matrix_comm_from_maps
 - AMGX_ZC_matrix_finalize is called

Similarly, vectors can be interfaced with in a zero-copy-manner too:

 - a vector is created as usual with AMGX_vector_create
 - the vectors data can be allocated/accessed with AMGX_ZC_vector_resize/get_ptr
 - after the matrix is finalized, vectors are bound with AMGX_ZC_vector_finalize_bind (instead of AMGX_vector_bind)

When solving for ZC right-hand side and solution vectors, AMGX_ZC_solver_solve replaces the combination of AMGX_vector_upload, AMGX_solver_solve and AMGX_vector_download. In particular, after AMX_ZC_solver_solve, the halo-entries of the solution-vector are consistent i.e. accessing them via the data-pointer gives the correct, global value.

The zero-copy interface also supports releasing the underlying data in AmgX matrices and vectors without entirely destroying these objects via AMGX_ZC_(matrix_data)_vec_resize/shrink_to_fit. This may be useful when matrices with the same sparsity structure are created repeatedly and the user wants to keep the GPU memory high-water mark down, the behavior of the AmgX memory pool can be queried and changed with AMGX_get/set_async_free_to_pool_flag to support this.
