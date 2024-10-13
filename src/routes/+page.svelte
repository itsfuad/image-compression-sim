<script lang="ts">
    import "$lib/style.scss";
    let imageFile: File | null = null;
    let N = 50;
    let canvas: HTMLCanvasElement;
    let ctx: CanvasRenderingContext2D | null;
    let imageUrl = "";
    let VerticalImages: { element: HTMLCanvasElement; url: string }[] = []; // To hold canvas elements and their URLs
    let FinalImages: { element: HTMLCanvasElement; url: string }[] = []; // To hold final images
    let isverticalCutCompleted = false; // To track if Step 2 is completed
    let stripWidth: number; // To hold the width of each strip from Step 2

    let error = "";

    let verticalRunning = false;
    let horizontalRunning = false;

    // Handle image upload and display the original image in the main canvas
    function handleImageUpload(event: Event) {
        const target = event.target as HTMLInputElement;
        if (target.files && target.files.length > 0) {
            imageFile = target.files[0];
            const reader = new FileReader();
            reader.onload = () => {
                imageUrl = reader.result as string;
                displayOriginalImage();
            };
            reader.readAsDataURL(imageFile);
        }
    }

    // Ensure N is even and a multiple of 10
    $: {
        if (N % 2 !== 0 || N % 10 !== 0) {
            N = Math.ceil(N / 10) * 10; // Adjust to nearest multiple of 10
        }
    }

    // Display the uploaded image on the main canvas
    function displayOriginalImage() {
        const img = new Image();
        img.onload = () => {
            canvas.width = img.width;
            canvas.height = img.height;
            ctx = canvas.getContext("2d");
            ctx?.drawImage(img, 0, 0);
            isverticalCutCompleted = false; // Reset the state
        };
        img.src = imageUrl;
        error = ""; // Clear any previous error
    }

    function NOk() {
        if (N % 2 !== 0) {
            error = "N must be an even number!";
            return false;
        }

        if (N < 50) {
            error = "N must be at least 50!";
            return false;
        }

        if (N > 600) {
            error = "N must be at most 600!";
            return false;
        }
        error = "";
        return true;
    }

    // Step 2: Split the image into N parts and merge odd and even strips
    function verticalStep() {
        if (!imageUrl) {
            error = "Please upload an image first!";
            return;
        }

        if (!NOk()) {
            return;
        }

        verticalRunning = true;

        const img = new Image();
        img.onload = () => {
            stripWidth = img.width / N; // Calculate strip width
            let output = splitImage(img); // Split image into N parts
            VerticalImages = groupOddsAndEvens(output, "x"); // Group odd and even strips
            isverticalCutCompleted = true; // Flag as Step 2 completed
            verticalRunning = false;
        };
        img.src = imageUrl;
    }

    // Split the image into N vertical parts
    function splitImage(img: HTMLImageElement) {
        let tempImages = []; // Clear previous images
        for (let i = 0; i < N; i++) {
            const stripCanvas = document.createElement("canvas");
            stripCanvas.width = stripWidth;
            stripCanvas.height = img.height;
            const stripCtx = stripCanvas.getContext("2d");
            stripCtx?.drawImage(
                canvas,
                i * stripWidth,
                0,
                stripWidth,
                img.height,
                0,
                0,
                stripWidth,
                img.height,
            );
            // Save canvas element and URL to render later
            tempImages.push({
                element: stripCanvas,
                url: stripCanvas.toDataURL(),
            });
        }
        return tempImages;
    }

    // Group odd and even strips and merge them into two new images
    function groupOddsAndEvens(
        images: { element: HTMLCanvasElement; url: string }[],
        mergeBy: "x" | "y",
    ) {
        const odds = images
            .filter((_, index) => index % 2 === 0)
            .map((img) => img.element);
        const evens = images
            .filter((_, index) => index % 2 === 1)
            .map((img) => img.element);
        return mergeImages(odds, evens, mergeBy);
    }

    // Merge odd and even strips into two new images
    function mergeImages(
        odds: HTMLCanvasElement[],
        evens: HTMLCanvasElement[],
        mergeBy: "x" | "y",
    ) {
        //merge by both x and y
        if (mergeBy === "x") {
            const newImage1 = document.createElement("canvas");
            const newImage2 = document.createElement("canvas");

            newImage1.width = odds[0].width * odds.length;
            newImage1.height = odds[0].height;

            newImage2.width = evens[0].width * evens.length;
            newImage2.height = evens[0].height;

            const ctx1 = newImage1.getContext("2d");
            const ctx2 = newImage2.getContext("2d");

            odds.forEach((image, i) =>
                ctx1?.drawImage(image, i * image.width, 0),
            );
            evens.forEach((image, i) =>
                ctx2?.drawImage(image, i * image.width, 0),
            );

            // Convert new canvases to URLs and store for rendering
            return [
                { element: newImage1, url: newImage1.toDataURL() },
                { element: newImage2, url: newImage2.toDataURL() },
            ];
        } else {
            const newImage1 = document.createElement("canvas");
            const newImage2 = document.createElement("canvas");

            newImage1.width = odds[0].width;
            newImage1.height = odds[0].height * odds.length;

            newImage2.width = evens[0].width;
            newImage2.height = evens[0].height * evens.length;

            const ctx1 = newImage1.getContext("2d");
            const ctx2 = newImage2.getContext("2d");

            odds.forEach((image, i) =>
                ctx1?.drawImage(image, 0, i * image.height),
            );
            evens.forEach((image, i) =>
                ctx2?.drawImage(image, 0, i * image.height),
            );

            // Convert new canvases to URLs and store for rendering
            return [
                { element: newImage1, url: newImage1.toDataURL() },
                { element: newImage2, url: newImage2.toDataURL() },
            ];
        }
    }

    // Step 3: Horizontally split each image into smaller strips of height equal to stripWidth
    function horizontalStep() {
        if (!NOk()) {
            return;
        }

        horizontalRunning = true;

        FinalImages = []; // Clear previous images
        let loadedImagesCount = 0; // To keep track of loaded images
        // Now we have two images in the VerticalImages array
        for (let i = 0; i < VerticalImages.length; i++) {
            const img = new Image();
            img.onload = () => {
                let output = splitImageHorizontally(img);
                output = groupOddsAndEvens(output, "y"); // Group odd and even strips
                FinalImages = [...FinalImages, ...output]; // Append to the final images
                loadedImagesCount++; // Increment the loaded image count
            };
            img.src = VerticalImages[i].url;
        }
        horizontalRunning = false;
    }

    // Split image horizontally into strips of height equal to stripWidth
    function splitImageHorizontally(img: HTMLImageElement) {
        const tempImages = [];
        const numberOfStrips = Math.ceil(img.height / stripWidth); // Calculate the number of horizontal strips

        for (let i = 0; i < numberOfStrips; i++) {
            const stripCanvas = document.createElement("canvas");
            stripCanvas.width = img.width;
            stripCanvas.height = Math.min(
                stripWidth,
                img.height - i * stripWidth,
            ); // Ensure height does not exceed original image height
            const stripCtx = stripCanvas.getContext("2d");
            stripCtx?.drawImage(
                img,
                0,
                i * stripWidth,
                img.width,
                stripWidth,
                0,
                0,
                img.width,
                stripCanvas.height, // Use the actual height of the strip
            );
            // Save canvas element and URL to render later
            tempImages.push({
                element: stripCanvas,
                url: stripCanvas.toDataURL(),
            });
        }
        return tempImages;
    }
</script>

<svelte:head>
    <title>Split Image Compression</title>
</svelte:head>

<h1>Split Image Compression</h1>
{#if error}
    <p class="error">{error}</p>
{/if}
<!-- Image upload input -->
<div class="upload-container">
    <input type="file" accept="image/*" on:change={handleImageUpload} />
    <label for="num-splits">Number of Vertical Strips (N):
        <input id="num-splits" type="number" max="600" bind:value={N} on:change={() => {
            NOk();
        }} />
    </label>
</div>

<button on:click={verticalStep} disabled={verticalRunning}>
    {#if verticalRunning}
        Processing...
    {:else}
        Vertical Cut
    {/if}
</button>

<!-- Main canvas for displaying the original image -->
<canvas bind:this={canvas}></canvas>

<div class="image-container">
    {#each VerticalImages as img}
        <img style="max-height: {canvas.getBoundingClientRect().height}px; width: {canvas.getBoundingClientRect().width / 2}px;" src={img.url} alt="Step 1 output" />
    {/each}
</div>

{#if isverticalCutCompleted}
<button on:click={horizontalStep} disabled={horizontalRunning}>
    {#if horizontalRunning}
        Processing...
    {:else}
        Horizontal Cut
    {/if}
</button>
{/if}

<div class="image-container final-images">
    {#each FinalImages as img}
        <img style="max-height: {canvas.getBoundingClientRect().height}px; width: {canvas.getBoundingClientRect().width}px;" src={img.url} alt="Final output" />
    {/each}
</div>

<style lang="scss">

    h1 {
        margin-bottom: 20px;
        font-size: 2em;
        color: #4a4a4a;
    }

    .upload-container {
        display: flex;
        align-items: center;
        gap: 15px;
        margin-bottom: 20px;
        flex-direction: row;
        flex-wrap: wrap;
    }

    input[type="file"],
    input[type="number"] {
        padding: 10px;
        border-radius: 5px;
        border: 1px solid #ccc;
        transition: border 0.3s;
    }

    input[type="file"]:hover,
    input[type="number"]:hover {
        border-color: #007bff;
    }

    button {
        padding: 10px 15px;
        border: none;
        border-radius: 5px;
        background-color: #007bff;
        color: white;
        cursor: pointer;
        transition: background-color 0.3s;
        margin-bottom: 10px;
    }

    button:hover {
        background-color: #1874d6;
    }

    .error {
        color: red;
        border-radius: 5px;
        border: 2px solid red;
        padding: 10px 20px;
        margin-bottom: 15px;
    }

    canvas {
        border: 1px solid #000;
        width: 100%;
        max-width: 400px;
        margin-bottom: 20px;
        box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
    }

    .image-container {
        display: flex;
        gap: 10px;
        margin-top: 20px;
        flex-direction: row;
        flex-wrap: wrap;
        justify-content: center;
        align-items: flex-start;
    }

    .final-images {
        margin-top: 40px;
    }

    img {
        max-width: 300px;
        border: 2px solid #007bff;
        border-radius: 5px;
        transition: transform 0.3s, border 0.3s;
    }

    img:hover {
        border-color: #0056b3;
    }
</style>
