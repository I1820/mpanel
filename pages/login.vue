<template>
</template>

<script>
export default {
  mounted () {
    this.$auth.loginWith('auth0')
  }
}
</script>
