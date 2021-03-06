<!-- markdownlint-disable MD002 MD041 -->

En este ejercicio, incorporará Microsoft Graph a la aplicación. Para esta aplicación, usará la [biblioteca de cliente de Microsoft Graph para .net](https://github.com/microsoftgraph/msgraph-sdk-dotnet) para realizar llamadas a Microsoft Graph.

## <a name="get-calendar-events-from-outlook"></a>Obtener eventos de calendario de Outlook

1. Agregue una nueva página para la vista de calendario. Haga clic con el botón secundario en el proyecto **GraphTutorial** en el explorador de soluciones y seleccione **Agregar > nuevo elemento..**.. Elija **página en blanco**, escriba `CalendarPage.xaml` en el campo **nombre** y seleccione **Agregar**.

1. Abra `CalendarPage.xaml` y agregue la siguiente línea dentro del `<Grid>` elemento existente.

    ```xaml
    <TextBlock x:Name="Events" TextWrapping="Wrap"/>
    ```

1. Abra `CalendarPage.xaml.cs` y agregue las siguientes `using` instrucciones en la parte superior del archivo.

    ```csharp
    using Microsoft.Graph;
    using Microsoft.Toolkit.Graph.Providers;
    using Microsoft.Toolkit.Uwp.UI.Controls;
    using Newtonsoft.Json;
    ```

1. Agregue las siguientes funciones a la `CalendarPage` clase.

    ```csharp
    private void ShowNotification(string message)
    {
        // Get the main page that contains the InAppNotification
        var mainPage = (Window.Current.Content as Frame).Content as MainPage;

        // Get the notification control
        var notification = mainPage.FindName("Notification") as InAppNotification;

        notification.Show(message);
    }

    protected override async void OnNavigatedTo(NavigationEventArgs e)
    {
        // Get the Graph client from the provider
        var graphClient = ProviderManager.Instance.GlobalProvider.Graph;

        try
        {
            // Get the user's mailbox settings to determine
            // their time zone
            var user = await graphClient.Me.Request()
                .Select(u => new { u.MailboxSettings })
                .GetAsync();

            var startOfWeek = GetUtcStartOfWeekInTimeZone(DateTime.Today, user.MailboxSettings.TimeZone);
            var endOfWeek = startOfWeek.AddDays(7);

            var queryOptions = new List<QueryOption>
            {
                new QueryOption("startDateTime", startOfWeek.ToString("o")),
                new QueryOption("endDateTime", endOfWeek.ToString("o"))
            };

            // Get the events
            var events = await graphClient.Me.CalendarView.Request(queryOptions)
                .Header("Prefer", $"outlook.timezone=\"{user.MailboxSettings.TimeZone}\"")
                .Select(ev => new
                {
                    ev.Subject,
                    ev.Organizer,
                    ev.Start,
                    ev.End
                })
                .OrderBy("start/dateTime")
                .Top(50)
                .GetAsync();

            // TEMPORARY: Show the results as JSON
            Events.Text = JsonConvert.SerializeObject(events.CurrentPage);
        }
        catch (ServiceException ex)
        {
            ShowNotification($"Exception getting events: {ex.Message}");
        }

        base.OnNavigatedTo(e);
    }

    private static DateTime GetUtcStartOfWeekInTimeZone(DateTime today, string timeZoneId)
    {
        TimeZoneInfo userTimeZone = TimeZoneInfo.FindSystemTimeZoneById(timeZoneId);

        // Assumes Sunday as first day of week
        int diff = System.DayOfWeek.Sunday - today.DayOfWeek;

        // create date as unspecified kind
        var unspecifiedStart = DateTime.SpecifyKind(today.AddDays(diff), DateTimeKind.Unspecified);

        // convert to UTC
        return TimeZoneInfo.ConvertTimeToUtc(unspecifiedStart, userTimeZone);
    }
    ```

    Tenga en cuenta lo que `OnNavigatedTo` hace el código.

    - La dirección URL a la que se llamará es `/me/calendarview` .
        - Los `startDateTime` `endDateTime` parámetros y definen el inicio y el final de la vista de calendario.
        - El `Prefer: outlook.timezone` encabezado hace que el `start` y `end` de los eventos se devuelvan en la zona horaria del usuario.
        - La `Select` función limita los campos devueltos para cada evento a solo aquellos que la aplicación usará realmente.
        - La `OrderBy` función ordena los resultados por fecha y hora de inicio.
        - La `Top` función solicita un máximo de 50 eventos.

1. Modifique el `NavView_ItemInvoked` método en el `MainPage.xaml.cs` archivo para reemplazar la `switch` instrucción existente por lo siguiente.

    ```csharp
    switch (invokedItem.ToLower())
    {
        case "new event":
            throw new NotImplementedException();
            break;
        case "calendar":
            RootFrame.Navigate(typeof(CalendarPage));
            break;
        case "home":
        default:
            RootFrame.Navigate(typeof(HomePage));
            break;
    }
    ```

Ahora puede ejecutar la aplicación, iniciar sesión y hacer clic en el elemento de navegación **calendario** en el menú de la izquierda. Debería ver un volcado JSON de los eventos en el calendario del usuario.

## <a name="display-the-results"></a>Mostrar los resultados

1. Reemplace todo el contenido de `CalendarPage.xaml` con lo siguiente.

    ```xaml
    <Page
        x:Class="GraphTutorial.CalendarPage"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:local="using:GraphTutorial"
        xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
        xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
        xmlns:controls="using:Microsoft.Toolkit.Uwp.UI.Controls"
        mc:Ignorable="d"
        Background="{ThemeResource ApplicationPageBackgroundThemeBrush}">

        <Grid>
            <controls:DataGrid x:Name="EventList" Grid.Row="1"
                    AutoGenerateColumns="False">
                <controls:DataGrid.Columns>
                    <controls:DataGridTextColumn
                            Header="Organizer"
                            Width="SizeToCells"
                            Binding="{Binding Organizer.EmailAddress.Name}"
                            FontSize="20" />
                    <controls:DataGridTextColumn
                            Header="Subject"
                            Width="SizeToCells"
                            Binding="{Binding Subject}"
                            FontSize="20" />
                    <controls:DataGridTextColumn
                            Header="Start"
                            Width="SizeToCells"
                            Binding="{Binding Start.DateTime}"
                            FontSize="20" />
                    <controls:DataGridTextColumn
                            Header="End"
                            Width="SizeToCells"
                            Binding="{Binding End.DateTime}"
                            FontSize="20" />
                </controls:DataGrid.Columns>
            </controls:DataGrid>
        </Grid>
    </Page>
    ```

1. Abra `CalendarPage.xaml.cs` y reemplace la `Events.Text = JsonConvert.SerializeObject(events.CurrentPage);` línea con lo siguiente.

    ```csharp
    EventList.ItemsSource = events.CurrentPage.ToList();
    ```

    Si ejecuta la aplicación ahora y selecciona el calendario, debe obtener una lista de eventos en una cuadrícula de datos. Sin embargo, los valores de **Inicio** y **finalización** se muestran de forma no fácil del usuario. Puede controlar cómo se muestran esos valores con un convertidor de [valores](https://docs.microsoft.com/uwp/api/Windows.UI.Xaml.Data.IValueConverter).

1. Haga clic con el botón derecho en el proyecto **GraphTutorial** en el explorador de soluciones y seleccione **Agregar > clase...**. Asigne un nombre `GraphDateTimeTimeZoneConverter.cs` a la clase y seleccione **Agregar**. Reemplace todo el contenido del archivo por lo siguiente.

    :::code language="csharp" source="../demo/GraphTutorial/GraphDateTimeTimeZoneConverter.cs" id="ConverterSnippet":::

    Este código toma la estructura [dateTimeTimeZone](/graph/api/resources/datetimetimezone?view=graph-rest-1.0) devuelta por Microsoft Graph y la analiza en un `DateTimeOffset` objeto. A continuación, convierte el valor en la zona horaria del usuario y devuelve el valor con formato.

1. Abra `CalendarPage.xaml` y agregue lo siguiente **antes** del `<Grid>` elemento.

    :::code language="xaml" source="../demo/GraphTutorial/CalendarPage.xaml" id="ResourcesSnippet":::

1. Reemplace los dos últimos `DataGridTextColumn` elementos por lo siguiente.

    :::code language="xaml" source="../demo/GraphTutorial/CalendarPage.xaml" id="BindingSnippet" highlight="4,9":::

1. Ejecute la aplicación, inicie sesión y haga clic en el elemento de navegación **calendario** . Debe ver la lista de eventos con los valores de **Inicio** y **finalización** con formato.

    ![Captura de pantalla de la tabla de eventos](./images/add-msgraph-01.png)
