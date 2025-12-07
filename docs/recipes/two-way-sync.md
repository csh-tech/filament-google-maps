# Two-Way Sync: Geocomplete & Map

A common requirement is to have a `Geocomplete` address field and a `Map` field that stay synchronized:
1. Selecting an address moves the map marker.
2. Dragging the map marker updates the address field.

To achieve this in Filament v3/v4 without conflicts, use the following configuration pattern.

## The Problem
If you naively enable `updateLatLng()` on the Geocomplete field and `autocomplete()`/`reverseGeocode()` on the Map field, they can conflict or cause infinite loops. Additionally, Filament's Livewire integration requires specific handling to prevent the map from being re-initialized (and flickering/disappearing) during updates.

## The Solution
Use the following setup:

1. **Geocomplete**: Updates `lat`/`lng` fields via `updateLatLng()`.
2. **Hidden Fields**: `latitude` & `longitude` store the coordinates.
3. **Reactive Map**: The hidden fields trigger a Livewire update (hooked via `afterStateUpdated`) to push the new location to the Map field.
4. **Map Reverse Geocoding**: The Map updates the address field when the marker is dragged.

### Code Example

```php
use Cheesegrits\FilamentGoogleMaps\Fields\Geocomplete;
use Cheesegrits\FilamentGoogleMaps\Fields\Map;
use Filament\Forms\Components\Hidden;
use Filament\Forms\Components\Section;

Section::make('Location')
    ->schema([
        Geocomplete::make('address')
            ->label('Address')
            ->updateLatLng() // Updates the hidden lat/lng fields
            ->reverseGeocode([
                'name' => '%n %S',
            ]),

        Map::make('location')
            ->label('Map Location')
            ->height('400px')
            ->defaultLocation([31.7857, 35.2007])
            ->draggable()
            ->clickable()
            ->autocomplete('address') // Follows the address field
            // Do NOT use autocompleteReverse(true) if you want to preserve the user's selected address text
            // ->autocompleteReverse(true) 
            ->reverseGeocode([
                'address' => '%n %S, %L', // Update address when marker is dragged
                'name' => '%n %S',
            ])
            ->columnSpanFull(),

        Hidden::make('latitude'),
        Hidden::make('longitude')
            ->live() // Trigger Livewire update when Geocomplete changes these
            ->afterStateUpdated(function ($state, callable $set, callable $get) {
                // Manually sync the Map's location state
                $lat = $get('latitude');
                if ($state && $lat) {
                    $set('location', [
                        'lat' => (float) $lat,
                        'lng' => (float) $state,
                    ]);
                }
            }),
    ])
```

## Key Configuration Points

1. **`Geocomplete::updateLatLng()`**: Essential. It pushes the selected place's coordinates to your hidden fields.
2. **`Map::autocomplete('address')`**: Tells the map to listen for changes on the address field input.
3. **`Map::reverseGeocode([...])`**: Configures the map to update your form fields (like `address`) when the marker is moved.
4. **`Map::autocompleteReverse(true)`**: **AVOID THIS** if you want "What I typed is what I get". If set to true, selecting "123 Main St" might immediately be reverse-geocoded by the map to "120-125 Main St" depending on where the pin lands, overwriting the user's selection.
5. **`live()` & `afterStateUpdated`**: This is the glue. When `Geocomplete` updates the hidden `longitude`, this hook fires and explicitly updates the `location` state of the Map field, ensuring the map center/marker moves to the new coordinates server-side (and then client-side).

## Filament v4 Note
Ensure your views are using the correct state path format (replacing `form.` with `data.` if necessary). This package includes fixes for this, but custom implementations should be aware.
