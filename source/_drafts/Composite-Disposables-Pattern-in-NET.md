title: Composite Disposables Pattern in .NET
tags:
  - .net
  - patterns
categories:
  - Code
date: 2016-01-27 10:34:00
---
# outline
0. problem with disposables that contain other disposables
   - once you need to pass disposable to other scope, proper disposal can get tricky
   - some implementations take a bool flag to dispose dependency (mostly streams) but not unbiquitous in framework
0. solved by building a CompositeDisposable class
0. new IDisposable implementations can take a params IDisposable[] in their constructor
   - params turned into CompositeDisposable and stored
   - when Disposed, composited disposable is disposed, too
0. is it too late for this pattern? the framework is already built
   - explore ways to adapt existing apis
   
   ---
   
   Some APIs involve creating objects that require cleanup at a later point in time. In .NET the thing to watch out for is whether a given class implements `IDisposable` and if it does, you really ought to call its `Dispose` method when your finished with it. C# and other CLR languages have language level support for guaranteeing that the `Dispose` method will be called:   

```c#
using(var someDisposableThing = new MyDisposableThingamajig())
{
	throw new ApplicationException("KABOOM!");
    
} // someDisposableThing.Dispose() invoked implicitely here :)
```
   
   This is all pretty common knowledge for any .NET developer. At least it should be. But sometimes we are forced to work with APIs that have us creating several of these disposable objects that will cooperate with each other to accomplish some task. 
   
   Look at this example:

```c#
using (var dataSet = new DataSet()) // 1
{
	using (var sqlConn = new SqlConnection(sqlConnString)) // 2
	{
		using (var sqlCmd = sqlConn.CreateCommand()) // 3
		{
			sqlCmd.CommandText = "SELECT * FROM Orders;";
			sqlConn.Open();
			
			using (var reader = sqlCmd.ExecuteReader()) // 4
			{
				var dataTable = new DataTable(); // 5
				dataTable.Load(reader);
				dataSet.Tables.Add(dataTable);
			}
			sqlConn.Close();
		}
	}
	
	DoSomethingReallyImportant(dataSet);
}

```

This code represents a complex interaction between the various classes required to query a SQL Server database. Several using blocks are present to ensure proper disposal, but the `DataTable` is not disposed by this code. Why not? Shouldn't it be? What's the harm if it's not disposed? Does disposing the `DataSet` that contains it implicitly dispose the `DataTable`? All valid questions and without intimate knowledge of the API it's hard to know what to do. Check out another example:

```c#
using (var file = File.OpenRead(@"c:\some\secret\file.dat")) // 1
using (var aes = new AesCryptoServiceProvider { Key = key, IV = iv }) //2
using (var decryptor = aes.CreateDecryptor()) // 3
using (var cryptoStream = new CryptoStream(file, decryptor, CryptoStreamMode.Read)) // 4
using (var tcpClient = new TcpClient("someHost", 1234)) // 5
using (var tcpStream = tcpClient.GetStream()) // 6
    cryptoStream.CopyTo(tcpStream);
```

This code is reading an encrypted file, decrypting the contents, and sending the cleartext to a TCP server. There are lots of interdependent objects working together and they're all disposable. This isn't a far-fetched thing to have to do, but the code is wildly oversimplified. In a real application you'd see interspersed code responsible for fetching the file location, host, and port from a configurable location. You'd probably see some key management code. Some logging code. Some error handling or retry code. Some business logic, if you can believe it!

What do developers tend to do when we find ourselves writing a mountain of code with many responsibilities? Some of us don't care. Others of of us worry over maintainability and clear comprehension of the code. This lot tries to decompose the system into reusable pieces that can be put back together in a clear and expressive way. 

Looking at these examples, it's tempting to just say, "No problem! I can make a class that will encapsulate all of this disposable state and their interactions. The class itself will be disposable and all will be made holy!"

Running with the last example, when encapsulating the code that decrypts the file, we might end up with something like:

```c#
public class HappyAesDecryptor : IDisposable
{
	readonly Aes _aes;
	readonly ICryptoTransform _dec;

	public HappyAesDecryptor(byte[] key, byte[] iv)
	{
		_aes = new AesCryptoServiceProvider {Key = key, IV = iv};
		_dec = _aes.CreateDecryptor();
	}

	public void Decrypt(Stream sourceCipherStream, Stream destClearStream)
	{
		using (var cryptoStream = new CryptoStream(sourceCipherStream, _dec, CryptoStreamMode.Read))
			cryptoStream.CopyTo(destClearStream);			
		
		// hmm… will disposing the cryptoStream also dispose the source stream?
		// that would be bad. we don't OWN that stream		
	}

	public void Dispose()
	{
		using(_aes)
			_dec.Dispose();
	}
}
```

That's just my first sorry attempt at encapsulating one piece of this puzzle. Though the class _does_ handle disposing the `Aes` and `ICryptoTransform` pair in it's own disposal logic, the disposal of the `CryptoStream` happens before hand. Worse, I'm not sure if disposing the `CryptoStream` disposes the source stream, which my class doesn't own. What if the caller isn't finished with it? I'm immediately not happy with this.

I could make all kinds of changes to rectify these problems, but there's darker forces working here. This code is attempting to abstract a highly flexible and feature rich API. When we create abstractions, we want to capitalize on the investment of doing so. We _really_ want to reuse the abstractions we've created. But by creating an abstraction flexibility ends up being lost. 

A contrived example, but maybe I want my `HappyAesDecryptor` to have the option of using the managed implementation of the AES algorithm. Or maybe any other symmetric encryption algorithm. Maybe I want to allow the Key/IV to be mutated after construction. I could build all of this configurability into my class to regain that lost flexibility. And once I've done that, I'd be left with a "leaky abstraction".

WIKIPEDIA BLOCK QUOTE

https://en.wikipedia.org/wiki/Leaky_abstraction

In software development, a leaky abstraction is an abstraction that exposes to its users details and limitations of its underlying implementation that should ideally be hidden away. Leaky abstractions are considered problematic, since the purpose of abstractions is to manage complexity by concealing unnecessary details from the use

I've wasted my time with building this thing!

I really want to take advantage of the flexible API I've been given but also be able to break things into pieces. Idisposable makes this difficult.... I want to pass one instance to something else and let it go on it's merry way.

todo finish this damn blog post