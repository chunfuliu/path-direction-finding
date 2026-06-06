<script>
	let {link} = $props();
</script>

<style>
a {
  color: black;
  padding: 12px 16px;
  text-decoration: none;
  display: block;
  font-family: Arial;
}	
	
a:hover {background-color: #ddd}
</style>


<a href="#{link.toLowerCase()}">{link}</a>
