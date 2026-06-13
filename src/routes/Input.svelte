<script>
    // Added a fallback empty string to $bindable
    let { inputValue = $bindable(""), placeholder = "Search..." } = $props();
</script>

<input 
    type="text" 
    {placeholder} 
    autocomplete="off" 
    class="search-input" 
    bind:value={inputValue} 
/>

<style>
    .search-input {
        box-sizing: border-box;
        font-size: 1.2rem;
        padding: 14px 20px 12px 20px;
        border: none;
        border-bottom: 1px solid #ddd;
        width: 100%;
        border-radius: 5px;
        
    }
    
    .search-input:focus {
        outline: 3px solid #f6bd60;

    }
</style>