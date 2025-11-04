<template>
<div class="spinner"></div>
</template>

<style>
.spinner {
  width: 40px;
  height: 40px;
  border: 4px solid transparent;
  border-top-color: #3cedbb;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}
</style>