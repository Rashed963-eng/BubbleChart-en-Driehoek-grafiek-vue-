<script setup>
</script>

<template>
    <nav>
      <div class="topnav">
        <img src="@/assets/img/SprocklerLogo.png" width="180" />
        <a class="active" href="#home">Home</a>
        <a href="#news">News</a>
        <a href="#contact">Contact</a>
        <div class="spacer"></div>
        <a href="#about">About</a>
      </div>
    </nav>
  </template>
  

    

<style>


.topnav {
    padding-top: 5px;
    padding-left: 5px;
    display:flex;
  background-color: hsl(0, 0%, 100%);
  overflow: hidden;
}

.topnav {
  padding: 5px;
  display: flex;
  align-items: center;
  background-color: hsl(0, 0%, 100%);
}

.topnav img {
  margin-right: 20px;
}

.topnav a {
  color: hsl(0, 0%, 0%);
  text-align: center;
  padding: 14px 16px;
  text-decoration: none;
  font-size: 17px;
}

.topnav a:hover {
  border-radius: 25px;
  background-color: hsl(216, 36%, 58%);
  color: #ffffff;
}

.topnav a.active {
  border-radius: 25px;
  background-color: hsl(215, 36%, 40%);
  color: #ffffff;
}

.spacer {
  flex: 1;
}


</style>