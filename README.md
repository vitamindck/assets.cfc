# Assets.cfc
Dynamically return a CSS/JS asset from a JSON manifest file for management of cachebusted filenames as generated by Gulp/Grunt build pipelines in ColdFusion applications.

# What does this do?
In modern web apps, we concatenate and minify our CSS and JS to improve the performance of our applications. For performance we enable long-lived caching headers to minimize repeat downloads. This requires us to change the filename each time we modify an asset (or else 'styles.css' won't be re-downloaded and the user won't see our changes if they've previously visited our site).  

We use Gulp, an awesome Javascript-based task runner, to handle the concatenation and minification. We also use a plugin gulp-rev to automatically rename the file based on the contents to give it a unique name like styles-j2l3zYq59kj.css.

The challenge, therefore, is how to get this always-changing filename into our markup for delivery to the user?  One option is gulp-inject which will update the HTML with the new CSS/JS filenames but this causes noisy commits. Instead, we created this very lightweight asset manager which reads a JSON file and translates from original to cachebusted name:

    <link rel="stylesheet" href="#asset.getAsset('style.css')#"> => <link rel="stylesheet" href="styles-j2l3zYq59kj.css">

# Generate rev-manifest.json

You can generate your manifest file many ways. We do it as part of a Gulp build pipeline that regenerates this file every time we modify a CSS or JS file. We have used https://www.npmjs.com/package/gulp-asset-manifest but currently use our own routine (in JS):

    app.createManifest = function(assets, ignorePath) {
    	var files = fs.readdirSync(assets);
    	var relpath = app.removeBasePath(ignorePath, assets);
    	var matches = {};
    
    	for (var ii = 0; ii < files.length; ii++) {
    		var f = files[ii];
    		var ext = path.extname(f);
    		if (ext == '.js' || ext == '.css') {
    			// gulp-rev generates files like base-name-file-0123456789.ext where the hash is always 10 characters - we want everything up to that last hyphen before the hash
    			matches[path.basename(f, ext).slice(0, -11) + ext] = relpath + f;
    		}
    	}
    
    	// the 4 here causes it to pretty print the file for readability 
    	fs.writeFileSync(assets + '/rev-manifest.json', JSON.stringify(matches, null, 4), 'utf-8');
    };

# Application.cfc

During init (e.g. onApplicationStart), create the asset manager and feed it your manifest file:

    <cfif NOT structKeyExists(application, "assets")>
        <cfset application.assets = createObject("component", "assets").init() />
    </cfif>
    <cfset application.assets.loadManifest(expandPath('./rev-manifest.json') />

Now in your HTML, you can output CSS/JS includes:

    <link rel="stylesheet" href="#application.assets.getAsset('style.css')#"> 
    <script src="#application.assets.getAsset('core.js')#"></script>
    
# Development Mode

During development, use a flag to reload the manifest file on each request to catch updated filenames in onRequestStart like:

    <cffunction name="onRequestStart" access="public" returntype="void" output="false">
		  <cfif TESTMODE>
			  <cfset loadAssetManifest() />
		  </cfif>
    </cffunction>


