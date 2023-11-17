---
title: Analyzing Stocks with .NET
draft: false
template: post
slug: "/analyzing-stocks-with-dotnet"
date: "2020-07-26T16:29:00Z"
category: "Markets"
social image: /media/BB_aapl.png
tags: 
    - ML
    - Stocks
    - Analysis
    - dotnet
description: "Studying historical price and volume data to forecast future stock price is called technical analysis. In this post, we'll look at some fundamental indicators, how to calculate them in .NET, and how those indicators can act as signals for us to consider trading."
---
![Bollinger Bands](/media/BB_aapl.png)

Year after year, hedge fund managers somehow managed to outperform their benchmark indices vastly and have even created profits in extreme market headwinds. How do they do it is a reasonable question to ask. There's a varied set of answers to that question. Much of it has to do with physical location and infrastructure. Part of it that I like to take solace in is that they do it largely with computational models and algorithms with which many programmers are familiar. They build models that analyze stocks at many levels and have programs that execute the trading strategies they've trained their models to implement. Most hedge fund Quantitative Analysts (Quants) have backgrounds in Computer Science, Math, Engineering, and Physics rather than traditional MBAs. And it's the skills they learned in those fields that are great value adds.

In this post, we'll be brushing the surface of these folks do. We'll learn to study historical price and volume data of stocks in an attempt to indicate future performance.

## Technical Analysis

Technical Analysis is the study of historical price and volume data of equity in an attempt to forecast what it will do in the future. Before we begin, I'll lay one big caveat. According to varieties of the [Efficient Market Hypothesis](https://en.wikipedia.org/wiki/Efficient-market_hypothesis), all information is baked into the price of a stock. So there's no edge to be had by studying historical data. Despite this, hedge funds persist in analyzing these trends, and consequentially to shall we!

## What's an Indicator

We'll be attempting to extract indicators from our historical price data, but what is an indicator? An indicator boils down to a number that "indicates" to us the direction of the price. Generally, we'll be looking for outliers in the price to determine whether there's a basis to assume the stock is "oversold" (less expensive than it should be) or "overbought"  (more expensive than it should be). If we detect those signals, we can move on the equity and potentially exploit an arbitrage in a slightly less than efficient market while the price of the stock corrects itself. We'll be looking at two such indicators - Simple Moving Average(SMA) Ratio, and Bollinger Bands® Percentage(BBP). We will also be looking at several other indicators used to calculate the SMA Ratio and the BBP. These indicators include the SMA, rolling volatility, and Bollinger Bands®.

## Create the Project

We'll dive right into the code now, so go ahead and create a .NET Core Console Application project and add the following NuGet packages to it:

```text
Daany.DataFrame
Numpy
ScottPlot
```

Python heavily inspires virtually everything we're going to do here. The NumPy library, which is what we're going to use to speed up our analysis, is dependant on a python runtime that ships with it.

For now create a folder called `data`, and files `BollingerBandsPercentage.cs`, `RollingVolatility.cs`, and `SimpleMovingAverage.cs`.

## Getting Historical Data

The key thing we'll need for technical analysis is historical stock data these data are readily available online at [Yahoo Finance](https://help.yahoo.com/kb/download-historical-data-yahoo-finance-sln2311.html?guccounter=1&guce_referrer=aHR0cHM6Ly93d3cuZ29vZ2xlLmNvbS8&guce_referrer_sig=AQAAABJ2C9mz-vMAEhjyvQOVenW_5SlmWHGAT1Jy-pOdyArjlkRVbHsua72K09hQ2mhY8IOccV8mAhnZdQjCf3Fp4PzWb4dbw56Q_ZprrdVM3NNPSh7NF9DHjhXwWRq_o6On4xJiiTp3vbVVlm07V8chJPNU3dKk0s9I2nVPudsdT0wW). It's a matter of searching for a particular equity, going to it's `Historical Data` tab, deciding on a period you'd like to grab records for and downloading it. These records come in Comma Separated Value (CSV) files. I'm going to get five years of this data for Apple (AAPL), JP Morgan Chase (JPM), and a major S&P 500 ETF (SPY). Add the three CSV files to your `data` folder. In Visual Studio, set the build action CSV to `Content` and the Copy to Output Directory to `Copy Always`.

## Reading in The Data

Now that we have all this historical stock data, we are going to read it in. We'll do this with the `DataFrame.FromCsv` method and store it in a `DataFrame` object. We'll also declare a constant int `WINDOW` which we're going to use as the window over which we'll be calculating our window averages.

```csharp
const int WINDOW = 20;
var symbol = "AAPL";
var df = DataFrame.FromCsv(Path.Join("data", $"{symbol}.csv"));
```

The df variable will now contain a dataframe, an in-memory representation of the `.csv` file. As we discussed earlier, there are two things we can study in technical analysis, Volume, and Price. These CSV files contain several prices, `Open`, `High`, `Low`, `Close`, `Adj Close`, and a representation of the `Volume` and `Date` for each date in the record.

Open is the price of the equity at the opening bell. High is the high highest price the equity traded for that day. Conversely, low is the lowest price the equity traded for that day.

Then there are two fields representing the closing price on the date. The "Close" is the price the stock traded for at the end of the day. And the Adjusted Close is the closing price adjusted for any stock splits or dividends. The Adjusted Close has to be updated whenever we hit one of those events. It's this `Adj Close` field that we will be paying the most attention to for these examples.

## Extracting Out the Adj Close

The `DataFrame` are represented more like dictionaries in C# as opposed to the traditional python Pandas. You can access the set of Adjusted Closes by indexing into the `DataFrame` on `Adj Close`. Then we can use a little bit of LINQ to pull the values into an array.

I'm going to store this in a NumPy array, slice the first 20 records out as that's our window size.

```csharp
var close = np.array(df["Adj Close"].Select(f => Convert.ToDouble(f)).ToArray()[rng])[$"{WINDOW - 1}:"];
```

Notice how we access the ranges inside the NumPy array? It looks a lot like traditional NumPy slicing notation, except it's using a string.

## Calculating Simple Moving Average

The simplest, pardon the pun, of the indicators we are going to look at is the Simple Moving Average. The SMA is, for each date in the data frame, the mean of the current point and a lookback window of the point. We will use a window of 20 trading days. Thus if we wanted to calculate the SMA for September 29th, 2015, we would take that day and the previous 19 trading days, sum them up, and divide them by 20. One of the cool things about a DataFrame structure is that it makes it easy to calculate these rolling products. I've defined a separate class to run this from `SimpleMovingAverage`, but the method itself is just a static `NDarray`.

```csharp
public class SimpleMovingAverage
{
    const int WINDOW = 20;
    public static NDarray CalculateSma(DataFrame df, Range rng)
    {
        return np.array(df.Rolling(WINDOW, Aggregation.Avg)["Adj Close"].Select(f => Convert.ToDouble(f)).ToArray()[rng])[$"{WINDOW - 1}:"];
    }
}
```

Pretty easy, right? Let's look at what we're doing here, we are calling `Rolling` on our dataframe, we are passing in a `WINDOW` of 20, and telling it to aggregate the average. Then we are accessing the `Adj Close` Column from our DataFrame and pulling that row out. We drop the first 20 rows from this because the first 20 rolls will be `NaN` as there wasn't a valid calculation.

## Calculating Rolling Volatility

Another metric that we want to look at is volatility. When we are thinking about equities in general, we consider high returns (the amount of money it makes) and a low risk to be ideal - low-risk high reward! We can calculate the volatility of a particular equity by finding its standard deviation from its rolling mean. This operation is similar to calculating the SMA, but for each point, we calculate the rolling standard deviation instead.

```csharp
public class RollingVolatility
{
    const int WINDOW = 20;
    public static NDarray CalculateRollingVolatility(DataFrame df, Range rng)
    {
        return np.array(df.Rolling(WINDOW, Aggregation.Std)["Adj Close"].Select(f => Convert.ToDouble(f)).ToArray()[rng])[$"{WINDOW - 1}:"];
    }
}
```

## Bollinger Bands®

BB's are easy to understand, and powerful indicator for seeing where the price of a stock is relative to where we'd expect it.

![Bollinger Bands for Apple](/media/BB_aapl.png)

In the graph above, we have three lines, an upper, lower, and a close. The upper band is the upper limit of the BB; the lower is the lower limit, and the close is where the stock ended up that day. When studying equities, it's threshold events that we tend to look at with indicators. In this instance, the criteria we are looking for are for the price of the stock to cross the upper or lower band. If we see it cross the upper band, it is generally a signal that the stock is overbought and primed to come back down to earth. When an equity is oversold, it can be a good time to [short the stock](https://www.investopedia.com/terms/s/shortselling.asp) (basically place a bet that it will go down). Conversely, if the equity sinks below it's lower Bollinger Band, it is considered oversold and primed to go up.

Calculating a Bollinger band is pretty straight forward after you've gotten the SMA and moving standard deviation. Simply multiply the standard deviation by two, then add the result to the SMA, and you have your upper band. Subtract that quantity, and you have the lower band. With what we've already built, this is a snap.

```csharp
var sma = SimpleMovingAverage.CalculateSma(df, rng);
var std2x = 2 * RollingVolatility.CalculateRollingVolatility(df, rng);
var upper = sma + std2x;
var lower = sma - std2x;
```

There you have it; the Bollinger bands are simply two standard deviations above and below the simple moving average.

Bollinger Bands aren't gospel and can certainly be wrong. For example - Tesla (TSLA) is at the moment killing it against it's Bollinger Bands. The losses you would have gotten hit with if you followed this strategy would have almost certainly created a margin call on your account - personally, with that level of risk and volatility in share price, I'm staying well away!

![Bollinger Bands for TSLA](/media/BB_TSLA.png)

## Bollinger Bands Percentage

Now that we have our Bollinger Bands, we're going to want an easily calculable number to serve as an indicator as to when we can move on an equity. Bollinger Bands Percentage gives us just that. We can calculate it by finding the difference between the price and the lower band, and then dividing that by the difference between the upper and lower band. If the result is north of one, it's crossed the upper band, below zero, and it's dropped below the lower band. With what we've built so far, calculating this is just one line of code.

```csharp
var bbp = (close - lower) / (upper - lower);
```

## Simple Moving Average Ratio

Another indicator we can use to look at whether an equity is overbought or oversold is to look at the SMA ratio. You can calculate this by dividing the price by the SMA. This indicator can also tell you whether the equity is oversold or overbought. The thresholds are bit murkier, I've heard .95 to 1.05 is normal anything above being overbought, anything below being oversold - but this is likely highly dependant on the equity. Regardless this is also easy to calculate given the SMA and the closing price:

```csharp
var smaRatio = close / sma;
```

## Displaying the Indicators

Now that we've calculated them, we're going to want to display them. We're going to be using [ScottPlot](https://github.com/swharden/ScottPlot), which provides some pretty nice plotting functionality. We're going to display these in scatter plots. These plots will look like lines because they are so dense. Get started by initializing a plot object and setting its size, generating a set of consecutive points with `DataGen.Consecutive` passing in the number of points that we have.

```csharp
var plt = new Plot(600, 400);
double[] xs = DataGen.Consecutive(bbp.size);
```

## Plot SMA Ratio

We'll start by plotting the SMA ratio. Add a title to the plot, then plot the `xs` points with the `smaRatio` points (call GetData on the NDArray to pull in all the data from it), pass it a label and set the marker size to 1.

Next, tell it to put the legend in the lower-left corner. Label the Y-axis `Ratio` and the X-axis `Day`, then save a png of the plot with the `SaveFig` function and clear out the plot.

```csharp
plt.Title($"SMA Ratio for {symbol} {startDate.ToShortDateString()} to {endDate.ToShortDateString()}");
plt.PlotScatter(xs, smaRatio.GetData<double>(), label: "SmaRatio", markerSize: 1);
plt.Legend(location: legendLocation.lowerLeft);
plt.YLabel("Ratio");
plt.XLabel("Day");
plt.SaveFig("SmaRatio.png");
plt.Clear();
```

The process for the other items is very similar. For the Bollinger Bands, you'll plot three different sets of data, and I found it easier to put the legend in the upper left.

## Plot Bollinger Bands

```csharp
plt.Title($"Bollinger Bands® for {symbol} {startDate.ToShortDateString()} to {endDate.ToShortDateString()}");
plt.PlotScatter(xs, close.GetData<double>(), label: "Close", markerSize: 1);
plt.PlotScatter(xs, upper.GetData<double>(), label: "Upper", markerSize: 1);
plt.PlotScatter(xs, lower.GetData<double>(), label: "Lower", markerSize: 1);
plt.YLabel("Dollars");
plt.XLabel("Day");
plt.Legend(location: legendLocation.upperLeft);
plt.SaveFig("BB.png");
plt.Clear();
```

## Plot Bollinger Band Percentage

```csharp
plt.Title($"Bollinger Bands® Percentage for {symbol} {startDate.ToShortDateString()} to {endDate.ToShortDateString()}");
plt.PlotScatter(xs, bbp.GetData<double>(), label: "BBP", markerSize: 1);
plt.YLabel("Dollars");
plt.XLabel("Day");
plt.Legend(location: legendLocation.lowerLeft);
plt.SaveFig("BBP.png");
plt.Clear();
```

After running these, plots will be written out to our bin directory looking something along the lines of these:

## BB.png

![Bollinger Bands](/media/BB_aapl.png)

## BBP.png

![Bollinger Bands Percentage](/media/BBP_AAPL.png)

## SmaRatio.png

![SMA Ratio](/media/SmaRatio_AAPL.png)

## Wrapping up

There are lots of other indicators that you can play with each with its merit. But these are some pretty simple ones you can use to whet your appetite. It's a fascinating field, and we're just scratching the surface of it. Obviously, consult a financial advisor before investing - but hopefully, this has provided you with a glimpse into how some trading strategies work!

## Next Steps

Now that we've started to create some indicators for us to look at, we can start thinking about using them in a trading strategy. Above, I listed some rules of thumb regarding the uses of these indicators in developing a strategy. Now that we have all these indicators organized so well, we can use them to create ML models to make those calls for us.

## Resources

* A list of technical indicators nad how to calculate them that I reference can be found [here](https://www.tradingtechnologies.com/xtrader-help/x-study/technical-indicator-definitions/list-of-technical-indicators/)
* The code that I used to build this can be found on [GitHub](https://github.com/slorello89/Indicators-dotnet)
